---
title: "Broken Resolution"
date: 2020-08-23T16:58:39-04:00
draft: false
---

# Breaking the resolution

Two machines on the internet can talk to each other iff both of them know each other's IP addresses.  When we type
in "google.com" in the browser,  the first task of browser is to get the IP address of "google.com" which it does
by communicating with DNS system. Once IP address is received the broswer establishes a TCP connection to the
server, does a TLS handshake, exchanges of data with the server before terminating the connection.

What would happen if DNS system gives a bad IP for a domain?  That is,  rather than giving `172.217.164.206` for
`google.com`,  DNS answers with some other IP. How can that happen? Apart from bug in DNS resolvers, there can be
cases where a malicious user might evesdrop for DNS queries from your system and replies back with IP address over
which she commands control and thereafter can snoop on the traffic from your machine.  

To check out this scenerio,  lets try to connect to `google.com` with an IP address of `example.com` such that TCP
and TLS handshake is done with `example.com`. There are multiple ways to do this: update your local
`/private/etc/hosts` to direct `google.com` to `example.com`,  inject the bad RR in locally running DNS resolver
[rrdns](https://github.com/lafolle/rrdns) or much simpler,  use curl's `--resolve` option.  

Lets go with 3rd option. 

We can create a small HTTPS server using SSL certs signed by our own root CA using
[mkcert](https://github.com/FiloSottile/mkcert).  Server code would execute the
following line to run TLS server given SSL certificate and private key:
```Go
http.ListenAndServeTLS(":443",
	"example.com+1.pem", // Certificate.
	"example.com+1-key.pem", nil) // Private key.
```
Check if the server is running correctly:
```bash
curl \
	--cacert rootCA.pem \
	--resolve example.com:443:127.0.0.1  \
	-vvv \
	https://example.com
```
Note, in the command:
1. curl is told about the new CA root added
2. curl is told to resolve `example.com` to 127.0.0.1 IP address and not use system resolver to do DNS resolution

Now lets resolve google.com to 127.0.0.1 and hit our test server:

```bash
curl \
	--cacert rootCA.pem \
	--resolve \
	google.com:443:127.0.0.1  \
	-vvv \
	https://google.com
```
which spits out

```bash {linenos=table, hl_lines=[12]}
* Added google.com:443:127.0.0.1 to DNS cache
* Hostname google.com was found in DNS cache
*   Trying 127.0.0.1:443...  [redacted - ALPN stuff]
* successfully set certificate verify locations:
*   CAfile: /Users/Karan/Library/Application Support/mkcert/rootCA.pem CApath: /usr/local/etc/openssl@1.1/certs
    [redacted - TLS 1.3 handshake stuff]
* Server certificate:
*  subject: O=mkcert development certificate; OU=Karan@Karans-MacBook-Air.local (Karan Chaudhary)
*  start date: Jun  1 00:00:00 2019 GMT
*  expire date: Aug 24 02:23:32 2030 GMT
*  subjectAltName does not match google.com
* SSL: no alternative certificate subject name matches target host name 'google.com' } [5 bytes data]
* TLSv1.3 (OUT), TLS alert, close notify (256): } [2 bytes data] curl: (60) SSL: no alternative certificate
  subject name matches target host name 'google.com' More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not establish a secure connection to it. To
learn more about this situation and how to fix it, please visit the web page mentioned above. 
```

We can see here that curl loaded its DNS cache with our entry,  tries and succeeds to establish a TCP connection
with localhost:443, loads the root certs from the custom path provided and then goes ahead to establish TLS 1.3
connection with the sever,  which fails because of _no alternative certificate subject name matches target host
name 'google.com'_.


Whats happened?

Every web server needs to prove its identity to the connecting client,  failure of which would
result in client terminating the connection.  The entity of proof is a digital certificate that web server offers
to the client, which on reception of cert runs multiple validations.  Who certifices the server? Server gets its
certifiaction from Certificate Authority (CA), a third party neutral organization whose job is to verify that the
server/org does own the domain for which it is asking the certificate for.

Server generates a public/private key pair and composes Certificate Signing Request (CSR) containing the public
key and name of domains for which it intends to get certificate for, signs it with its private key and sends it to
CA.  CA verifies the CSR using public key in it and also verifies if the domains for which the certifcate is
requested for does actually belong the requesting org/server/entity.  Once verification succeeds,  CA generates
the X.509 certificate containing public key of the requesting org, for how long the certificate is valid,
identitiy of itself as to who generated the certificate and for which domains this certificate is valid for
and other various fields.

Note from the section above that the certificate is valid for selected list of domains.  When client made a
connection with server,  the server sent back its certificate to affirm its identity.  Apart from many other
validations (like is the certificate signed by a verified CA, is the cert expired or not,  etc) the client also
checks if the domain name to which it was trying to connect to matches the CN (Common Name) or SAN (Subject
Alternate Name) present in the certificate.  This is the stuff that failed in our case as neither CN nor SAN
in certificate matches `google.com`.  What are they in the certificate?  CN is set to "mkcert
<username>@<machine-name>.local (<my full name>)" and SAN consists of DNS name=example.com and IP Address =
127.0.0.1. Both of these were set by the `mkcert` command for us.


Cool, so we now know why the failure occured.  But what if we generate the certificate where SAN matches
"google.com"?

Lets generate cert for google.com: `mkcert google.com`,  and load them into the server such that server looks like
this:
```Go
http.ListenAndServeTLS(":443",
	"google.com.pem", // Certificate.
	"google.com-key.pem", nil) // Private key.
```

 and hitting the server with following curl request:
 ```bash
curl \
	--cacert rootCA.pem \
	--resolve google.com:443:127.0.0.1  \
	-vvv \
	https://google.com
 ```
returns 200
```bash {linenos=table, hl_lines=[6]}
[redacted]
* Server certificate:
*  subject: O=mkcert development certificate; OU=Karan@Karans-MacBook-Air.local (Karan Chaudhary)
*  start date: Jun  1 00:00:00 2019 GMT
*  expire date: Aug 24 20:38:06 2030 GMT
*  subjectAltName: host "google.com" matched cert's "google.com"
*  issuer: O=mkcert development CA; OU=Karan@Karans-MacBook-Air.local (Karan Chaudhary); CN=mkcert Karan@Karans-MacBook-Air.local (Karan Chaudhary)
*  SSL certificate verify ok.
[redacted]
```

This is expected.  The certificate does have `google.com` in it under SAN section which
matches domain present in the request.

One big advantage we have over an attacker is that we are able to inject CA's root certificates into the client.
Note the "--cacert" option passed to `curl` which tells `curl` where to find the root CA certificate which was
used to issue X.509 cert to server.  This ability is denied to attacker.  Mostly clients come bundled with CA certs
and can also be provisioned to use CA certs installed in system.

Is it possible for an attacker to use `google.com`s certificate as anyways it is public?

<!-- # Using google.com's cert -->

How can we get the SSL cert for `google.com` in PEM format??? Easy:
```bash
echo | \
	openssl s_connect -showcerts -connect google.com.443 \
	openssl x509 -out google.com-original.pem
```
This will create a file `google.com-original.pem` that would contain the digital certificate for `google.com`.  We
can add it to our server  but there is a problem.  We don't have the private key.  In fact, if we can have the
corresponding private key,  Google's existance is literally doomed.

This is a blocker but lets go forward and try to feed-in wrong private key to the Go server:
```Go
http.ListenAndServeTLS(":443",
	"google.com-original.pem", // Certificate.
	"google-key.pem", nil) // Private key.
}
```

On running, the server fails with following error:
```bash
listeing on localhost:443
2020/08/24 17:42:44 ListenAndServe: tls: private key does not match public key
exit status 1
```

Of course!  That is expected.  Now we are certain that attacker cannot do any damage unless they have the private
key corresponding to the public key.

If the imposter is somehow able to direct the browser to her IP which is backed by a webserver also controlled by
her,  unless the website is being accessed by HTTPS,  she won't be able to do any harm because the whole
certificate dance prevents her from faking the identity. But it is important to note that this only works if the
website is being accessed over HTTPS.  In case the website is accessed over HTTP,  there is no way for the client
to verify the identity of server it is communicating with and is easy for the attacker to fake its identity.  For
example,  she might direct `scotiabank.com` (a major bank in Canada) to her own malicious version of the bank's
website and ignorant user won't be able to differentiate and might end up giving private info to the attacker, if
the servers running the site were operating over HTTP.

Though browsers are not unaware of this case and can take steps to inform the user about such insecurity. For
example,  in Firefox,  accessing a website over HTTP, `http://www.bccancer.bc.ca/`,  (some site do a redirection
from 80 -> 443) will result in indication on the url bar highlighting that communication with website is insecure
via a red mark on a lock.  Though that is true enough to alert the user about the pitfalls of using the website,
the truth is,  i can't even verify if this website truly belongs to `bccancer.bc.ca`.

![Firefox warning](/broken-resolution/firefox-warning.png)

Another role browsers play is to enforce HTTPS on certain domains using something called HSTS policy mechanism.
It states that if when communicating with website W there comes a response with header containing HSTS
(Strict-Transport-Security) record then all future calls to W will be made over HTTPS only until the duration
specified in HSTS header (`max-age` directive).  What about sub-domains like x.W or y.x.W? This is handled by
another valueless directive in the header named "includeSubDomains" that "signals the UA [User Agent eg browser]
that the HSTS Policy applies to this HSTS Host as well as any subdomains of the host's domain name."

But then what would happen if the user is accessing the website E for the first time or when the HSTS policy has
been expired,  because in that case HSTS policy won't kick in as browser has never communicated with E earlier or
there is no valid HSTS policy in browser? Lo and behold, all major browsers come up with a pre-loaded list of
websites for which HSTS paramaters have been submitted before hand by the website owners.

# Conclusion

Private keys need to be kept secret.  Losing them can jeopardize whole traffic originating for the domain.
Digital certificates are really important to asertain the identity of server apart from playing an important role
in keeping the traffic encrypted.

