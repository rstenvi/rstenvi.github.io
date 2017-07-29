---
layout: post
title: DNS Rebinding
---

# Motivation

Several embedded systems are weakly configured and almost completely open when
inside the internal network. DNS rebinding can serve an effective role her in
sending network requests from inside the network. I was not satisfied with the
existing tools for DNS rebinding, so I decided to create my own and test current
protections against DNS rebinding.

The source code for DNSRebinder is at Github: [DNSRebinder](https://github.com/rstenvi/DNSRebinder/).

# DNS Rebinding Theory

DNS rebinding is an attack that violates the Same-Origin-Policy (SOP) protection
browsers implement (Adobe, Java and others also implement SOP, but is not discussed
here). The attack is most useful against internal systems that are accessed
using IP and not domain name. 

I will not describe the attack in detail here as that is done several other
places. I will just provide a short outline of DNS rebinding attack:

1. A client visits a malicious web site, ```attacker.com```. The attacker's DNS server
resolves ```attacker.com``` to the attacker's web server which includes a JavaScript
payload. The DNS server is also configured with a short Time-To-Live (TTL).
1. After the payload has been delivered to the victim's browser, the attacker
rebinds ```attacker.com``` to the target's IP address.
1. After some time, the malicious JavaScript connects to ```attacker.com``` and because of the short
DNS TTL, the browser will perform a new DNS request for ```attacker.com``` and receive
the target's IP address. In addition to DNS TTL, most browsers will implement DNS pinning and therefore the
timeout probably have to be longer than the DNS TTL.
1. Because SOP is based on hostname, the malicious JavaScript can now read
responses from the target's IP address.

Although any IP address can be used as a target for DNS rebinding, the attack is
most useful for targeting internal IP addresses. Below are some pre-requisites
for the attack.

1. The attacker must somehow get the victim to visit a domain where the attacker
controls the authoritative name server. 
1. Visit to the web site could happen through any means, including injection of
iframes in legitimate sites.
1. Requests to the internal web server cannot modify any of the forbidden HTTP
headers[1]. The most significant forbidden
header is the 'Host'-header. If the web server we are targeting checks the
Host-header, the web server will see that we come from a different domain and
possibly reject the request.

# DNS Rebinding Implementation

The key part of the attack is to change the DNS entry after the payload has been
loaded in the victim's browser. This means that we must respond differently
depending on whether the payload has been delivered or not. As the client will
often ask an external DNS server to perform the DNS query, we cannot correlate
IP address from web request and IP address from DNS query. Instead, a random
subdomain is generated for each client. The JavaScript loaded in the client's
browser can then notify web the server about when the DNS entry should be changed.

This method has two benefits:

1. It allows arbitrary many clients at the same time.
1. We can attack arbitrary many internal IPs at each client. To accomplish this,
the first payload will generate a series of iframes, each with its own payload
and each DNS entry will be changed to its own internal IP address.

# DNS browser behaviour

During the development, I also performed a test of how different browsers
implemented DNS pinning and how long the victim must have the site opened. As
the behaviour for Internet Explorer and Edge are different from the others, two
different methods are used.

## Method 1

The first method is completely straight-forward and works in most browsers
within a reasonable timeframe.

### Description of testing method

Below is an overview of the testing methodology:

1. Visit ```http://example.com/redirect```
1. Server redirects the user to a random subdomain
(```http://qwerty.a.example.com/```)
1. The DNS server resolves ```qwerty.a.example.com``` to the IP that has the payload,
i.e. the external web server.
1. The JavaScript payload will notify the web server that it has loaded and that the
DNS entry should be modified.
1. The server modifies the DNS entry to a local IP, i.e.
```qwerty.a.example.com``` is
modified to IP-address ```192.168.0.10```. During testing this local IP ran a web
server to verify when the IP changed.
1. Every 5 second, JavaScript will attempt to contact the web server it has been
loaded on (```qwerty.a.example.com```).
1. When an invalid response has been recieved, the JavaScript will stop. The
internal web server can then be checked to verify that the last request was
performed against the internal IP address.


### Results

This method of DNS rebinding worked for most browsers within a reasonable
timeframe. Below are the results for the browsers tested:


| Browser       | Seconds   | Comments  |
|:------------- |:---------:| :---------|
| Firefox       | 65        | New DNS lookup is done immediately after 60 seconds has passed |
| Chrome        | 60        | |
| Opera         | 60        | |
| Vivaldi       | 60        | |

As can be seen, most browsers can be attacked in about a minute. A successful
attack therefore requires that the user has the tab open for a minute.

## Method 2

The method described above did not work for IE and Edge within a reasonable
timeframe. After having the tab open for 30 minutes and still no change, I
decided to use a different approach.

The approach relies on having two DNS A-records for the domain being queried.
The first A-record is the real IP and the second A-record is the internal IP being
targeted. The browser then has a choice and can choose either IP. This was tested on
all the browsers and below are the result of which IP was chosen on first visit:

| Browser       | Result      |
|:------------- |------------:|
| Chrome        | External IP |
| Firefox       | Internal IP |
| Opera         | Internal IP |
| Vivaldi       | Internal IP |
| IE            | External IP |
| Edge          | External IP |

If the internal IP is chosen we will not be able to deliver the payload and
therefore the attack will be lost.


Since some browsers chooses the internal IP we cannot use this method
immediately. If we were to do that, we would fail to attack Firefox, Opera and
Vivaldi. Instead, this attack can be loaded after we have delivered the initial
payload and determined what browser they are using. A new attack can be loaded
by inserting a new iframe.

### Description of testing method

The steps are described below:

1. User visit ```http://example.com/redirect```
1. Server redirects the user to a random subdomain
```http://qwerty.a.example.com/```
1. The DNS server resolves ```qwerty.a.example.com``` to the IP that has the payload
1. JavaScript determines that the browser is IE or Edge
1. JavaScript creates a new iframe pointing to
```http://example.com/redirect?sub=b```
1. Web server redirect the previous URL to a new random subdomain:
```asdfgh.b.example.com```.
1. Browser retrieves the DNS record for: ```asdfgh.b.example.com``` - has two DNS
A-records (external IP and internal IP)
1. Browser will use the first (external IP) and retrieve the payload from the
server
1. JavaScript on the new domain (```asdfgh.b.example.com```) will instruct the server
to deny HTTP traffic from this client
1. Server creates an iptables rule blocking the client from accessing port 80 on
the server
1. JavaScript attempts to insert an image from the domain
(```asdfgh.b.example.com```), but will fail because of the previous rule
1. Every second the JavaScript will attempt to contact the web server at
```asdfgh.b.example.com``` and stop when the private IP address is used.

#### Notes

A few notes on this attack below:

1. Although this attack also works for Chrome, no time is saved.
1. The technique iframes is also useful for targeting several internal IP
addresses. Each iframe is using its own subdomain and subject to its own DNS pinning
timeout. Therefore, several IPs can be targeted with 1 visit from the user and
directly after the DNS timout period.
1. I have not changed anything on the settings in the browser.
1. I tried several times on each browser and always received the same result, so
I don't think any of the browsers choose IP-address at random (this includes all
5 browsers tested).


### Results

Below are the results of the test, Internet Explorer and Edge can now be
attacked much quicker.

| Browser       | Seconds   | Comments  |
|:------------- |:---------:| :---------|
| IE            | 1         | Second request made |
| Edge          | 0         | First request made  |

# Conclusion

DNS Rebinding is still an effective attack against internal systems. The biggest
problem for the attacker is that they are usually blind to which systems exist
on the internal network.

# Footnotes

[1] [https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name)
