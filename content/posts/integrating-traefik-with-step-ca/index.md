---
title: "Securing Docker Traffic by Integrating Traefik With step-ca"
date: 2024-11-04T00:21:12-05:00
draft: false
toc: true
---

## Introduction

At work, we use Traefik to proxy all of our production internet-facing services. One super convenient aspect of this is that Traefik automatically integrates with Let's Encrypt to get and update certificates for all of these services, taking away what would otherwise be a massive certificate management challenge. I wanted to get the same convenience for my home Docker environment, but Let's Encrypt requires that your DNS be public to verify requests. I do not have any internet-facing services and did not want to use a public domain for my homelab. Luckily, we can achieve the same thing using the `step-ca` online Certificate Authority and a private domain!

This post is built upon a lot of prior work.

* In our post [Setting Up a Traefik Application Proxy for Docker](/posts/setting-up-a-traefik-reverse-proxy-for-docker/) we set up a basic Traefik install to proxy traffic to our Docker services.
* In [Setting Up a Private Domain with Technitium](/posts/private-domain/) we created a private `unicorn.home` domain for our homelab.
* Finally, in [step-ca certificate authority for our private domain](/posts/private-ca-with-step-ca/) we created a private Certificate Authority to issue certs for our private domain.

Today we'll integrate these pieces to add secure HTTPS connections and automated certificate management to our Docker environment.

Our goals today are:

1. Get all traffic going to Traefik to use HTTPS.
2. Integrate Traefik and step-ca using the ACME protocol to issue and renew certificates automatically.

## Securing Traffic to Docker-based Services by Using SSL Between the User and Traefik

First, lets get HTTPS working between the user and Traefik. There are a few steps to doing so, let's walk through them.

### Creating a Secure Traefik Entrypoint

This section assumes you've already got Traefik installed and working for HTTP requests. If not, see [Setting Up a Traefik Application Proxy for Docker](/posts/setting-up-a-traefik-reverse-proxy-for-docker/).

First, let's open up a secure HTTPS pathway through Traefik. We'll have to do two things:

1. Create a new entryPoint in Traefik's static config.
2. Publish port 443 from Traefik to the docker host.

We'll start by creating a new entryPoint on our Traefik static configuration. If you're using a traefik.yml file, add this to your entryPoint configuration: 

```yaml
entryPoints:
  web:
    address: ":80"
  secureweb:
    address: ":443"
```

If you're using command line arguments or docker compose, pass this additional argument: `"--entryPoints.secureweb.address=:443"`. The `secureweb` is just the name we've given the entryPoint, there's nothing magical about that name.

Next, make sure that your Traefik container is publishing port 443 to the docker host so it can receive https traffic. Do that in a docker compose like this:

```yaml
services:
  traefik:
    image: "traefik:v3.2"
    container_name: "traefik"
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entryPoints.web.address=:80"
      - "--entryPoints.secureweb.address=:443"
      - "--accesslog=true"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - "//var/run/docker.sock://var/run/docker.sock:ro"
```

or add a `-p 443:443` to your docker run command line.

Restart Traefik, and a visit to the dashboard should show the new :443 entrypoint.

### Configuring Services to Use the Secure EntryPoint

We need to configure a new router on our service(s) to utilize this new secure path.

```yaml
  whoami:
    image: traefik/whoami
    command:
       - --name=iamfoo
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.dockerhost.local`)"
      - "traefik.http.routers.whoami.entrypoints=web"
      - "traefik.http.routers.whoamisecure.entrypoints=secureweb"
      - "traefik.http.routers.whoamisecure.rule=Host(`whoami.dockerhost.local`)"
      - "traefik.http.routers.whoamisecure.tls=true"
```

Here we're re-using our sample `whoami` service from our earlier post. I've added three new labels, the last three on the list. I'm creating a new router called `whoamisecure` and mapping it to the `secureweb` entryPoint. I'm creating the rule to route `whoami.dockerhost.local` to this router. And finally, I'm enabling TLS on this router. If some of this configuration feels a bit redundant or repetitive, don't worry - we're just going through an intermediate state here, we'll clean this up in a bit.

In this configuration, Traefik will terminate TLS itself and use plain HTTP to talk to the back-end service, so there's no need for the actual service to support TLS/HTTPS connections. This is a great way to protect services that don't have HTTPS by default!

Let's demonstrate how this configuration works. Restart Traefik and our demo `whoami` service, and then try to reach it via https. We'll need to skip certificate verification for now (if you do this in a browser, expect a certificate warning). I'm turning on curl's verbose mode so we can see the TLS and certificate information.

```
$ curl --insecure -v https://whoami.dockerhost.local/
*   Trying 127.0.0.1:443...
* Connected to whoami.dockerhost.local (127.0.0.1) port 443 (#0)
[SNIP UNRELATED OUTPUT]
* Server certificate:
*  subject: CN=TRAEFIK DEFAULT CERT
*  start date: Nov  4 16:35:53 2024 GMT
*  expire date: Nov  4 16:35:53 2025 GMT
*  issuer: CN=TRAEFIK DEFAULT CERT
*  SSL certificate verify result: self-signed certificate (18), continuing anyway.
[SNIP UNRELATED OUTPUT]
Name: iamfoo
Hostname: 93b894e2cab3
IP: 127.0.0.1
IP: ::1
IP: 172.18.0.2
RemoteAddr: 172.18.0.3:56636
GET / HTTP/1.1
Host: whoami.dockerhost.local
User-Agent: curl/7.81.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.dockerhost.local
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: 647aa0ca55fc
X-Real-Ip: 172.18.0.1

* Connection #0 to host whoami.dockerhost.local left intact
```

There we go, we've used https to reach our `whoami` service. We can see that Traefik used a self-signed certificate to encrypt the connection (`issuer: CN=TRAEFIK DEFAULT CERT`), which is why we needed the `--insecure` flag on curl. Notice that the communication to the backend service was still HTTP: `GET / HTTP/1.1`.

In this configuration, plain HTTP still works as well:

```
$ curl http://whoami.dockerhost.local/
Name: iamfoo
Hostname: 93b894e2cab3
IP: 127.0.0.1
IP: ::1
IP: 172.18.0.2
RemoteAddr: 172.18.0.3:45302
GET / HTTP/1.1
Host: whoami.dockerhost.local
User-Agent: curl/7.81.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.dockerhost.local
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 647aa0ca55fc
X-Real-Ip: 172.18.0.1
```

### Redirecting all Requests to the Secure EntryPoint

Traefik is capable of redirecting all traffic from the HTTP endpoint to the HTTPS endpoint. That's a nice way to keep http and https URLs working while enforcing secure connections, so let's do that.

Via docker compose or docker command-line, you add two additional parameters to the command, `--entrypoints.web.http.redirections.entryPoint.to=secureweb` and `--entrypoints.web.http.redirections.entryPoint.scheme=https`.

```yaml
  traefik:
    image: "traefik:v3.2"
    container_name: "traefik"
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entryPoints.web.address=:80"
      - "--entryPoints.secureweb.address=:443"
      - "--entrypoints.web.http.redirections.entryPoint.to=secureweb"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--accesslog=true"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - "//var/run/docker.sock://var/run/docker.sock:ro"
```

Alternately, in your `traefik.yml` file:

```yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https

  secureweb:
    address: ":443"
```

Let's restart Traefik and test again with curl:

```
$ curl --insecure http://whoami.dockerhost.local/
Moved Permanently
```

Ah, curl is receiving the redirect (`Moved Permanently`) but doesn't follow redirects by default. Let's tell it to do so with the `-L` flag.

```
$ curl --insecure -v -L http://whoami.dockerhost.local/
*   Trying 127.0.0.1:80...
* Connected to whoami.dockerhost.local (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: whoami.dockerhost.local
> User-Agent: curl/7.81.0
> Accept: */*
>

[SNIP]

< HTTP/1.1 301 Moved Permanently
< Location: https://whoami.dockerhost.local/
< Date: Mon, 04 Nov 2024 19:49:46 GMT
< Content-Length: 17

[SNIP]

* Clear auth, redirects to port from 80 to 443
* Issue another request to this URL: 'https://whoami.dockerhost.local/'
*   Trying 127.0.0.1:443...
* Connected to whoami.dockerhost.local (127.0.0.1) port 443 (#1)

[SNIP]

* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=TRAEFIK DEFAULT CERT
*  start date: Nov  4 19:46:50 2024 GMT
*  expire date: Nov  4 19:46:50 2025 GMT
*  issuer: CN=TRAEFIK DEFAULT CERT
*  SSL certificate verify result: self-signed certificate (18), continuing anyway.

[SNIP]

> GET / HTTP/2
> Host: whoami.dockerhost.local
> user-agent: curl/7.81.0
> accept: */*
>

[SNIP]

< HTTP/2 200
< content-type: text/plain; charset=utf-8
< date: Mon, 04 Nov 2024 19:49:46 GMT
< content-length: 390

[SNIP]

Name: iamfoo
Hostname: 64bb81d24164
IP: 127.0.0.1
IP: ::1
IP: 172.18.0.2
RemoteAddr: 172.18.0.3:45294
GET / HTTP/1.1
Host: whoami.dockerhost.local
User-Agent: curl/7.81.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.dockerhost.local
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: c3549b275125
X-Real-Ip: 172.18.0.1
```

I've turned on verbose output but snipped out a lot of irrelevant content so we can clearly see that the request was redirected and used an SSL connection despite us using the plain http URL.

Now that we're redirecting all traffic to the secureweb router at the Traefik level, the `whoami` service no longer needs a plain http router. Let's clean up its config.

```yaml
  whoami:
    image: traefik/whoami
    command:
       - --name=iamfoo
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoamisecure.entrypoints=secureweb"
      - "traefik.http.routers.whoamisecure.rule=Host(`whoami.dockerhost.local`)"
      - "traefik.http.routers.whoamisecure.tls=true"
```

Here I've removed the two labels for the non-secure whoami router, leaving just the whoamisecure router. Restart the `whoami` container. Because redirection is happening at the Traefik level, we can still use both http and https URLs to reach `whoami`.

```
$ curl --insecure -L http://whoami.dockerhost.local/
Name: iamfoo
Hostname: a78e6cfe05b0
IP: 127.0.0.1
IP: ::1
IP: 172.18.0.2
RemoteAddr: 172.18.0.3:50322
GET / HTTP/1.1
Host: whoami.dockerhost.local
User-Agent: curl/7.81.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.dockerhost.local
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: 3774eacc3137
X-Real-Ip: 172.18.0.1

$ curl --insecure https://whoami.dockerhost.local/
Name: iamfoo
Hostname: a78e6cfe05b0
IP: 127.0.0.1
IP: ::1
IP: 172.18.0.2
RemoteAddr: 172.18.0.3:50322
GET / HTTP/1.1
Host: whoami.dockerhost.local
User-Agent: curl/7.81.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.dockerhost.local
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: 3774eacc3137
X-Real-Ip: 172.18.0.1
```

Now we have secure connections to our services (or at least from the end user to Traefik), regardless of the URL the user uses, and without the service itself needing to support https connections.

## Using the ACME Protocol to Automatically Get and Renew Certificates

Now we get to the point where we can really make a useful and powerful solution. This section assumes you've already got a step-ca Certificate Authority running and a domain name for your homelab. If not, you can reference [Setting Up a Private Domain with Technitium](https://franfabrizio.dev/posts/private-domain/) and [Setting up a step-ca certificate authority for our private domain](/posts/private-ca-with-step-ca/).

To accomplish our goal, we will need to:

1. Add an ACME provisioner to step-ca
2. Add step-ca as a certificate resolver to Traefik.
3. Make sure that Traefik trusts your CA.
4. Add labels to containers to enable Traefik to manage certificates for them.

### Add an ACME Provisioner to step-ca

We first need to enable ACME on our step-ca certificate authority. Use the step client to do so:

```
$ step ca provisioner add acme --type=ACME
```

This will create a new provisioner for ACME requests. Verify that it worked with:

```
$ step ca provisioner list

[

[SNIP]

   {
      "type": "ACME",
      "name": "acme",
      "claims": {
         "enableSSHCA": true,
         "disableRenewal": false,
         "allowRenewalAfterExpiry": false,
         "disableSmallstepExtensions": false
      },
      "options": {
         "x509": {},
         "ssh": {}
      }
   }
]
```

Make sure the new acme provisioner is in the list.

### Configuring step-ca as a Certificate Provider

Next we need to extend Traefik's static configuration to add a certificate resolver. Here's an example using `traefik.yml`.

```yaml
certificatesResolvers:
  stepca:
    acme:
      caServer: "https://<YOUR-STEP-CA-SERVER>:9000/acme/acme/directory"
      email: "YOUR@EMAIL.ADDRESS"
      certificatesDuration: 2160
      httpChallenge:
        entryPoint: http
```

* **caServer** is the URL to your online step-ca Certificate Authority which we [set up in an earlier post](/posts/private-ca-with-step-ca/).
* The **httpChallenge** section is in relation to the ACME protocol. The way that ACME verifies that a request for a certificate is legitimate is to challenge the requestor to prove who they are. One way to do so is via an HTTP challenge. An ACME HTTP challenge requires the requestor to host a random number at a random URL on port 80. This section helps Traefik configure those challenges.

If you're using docker compose/CLI to configure Traefik, this is what the command parameters look like:

```yaml
      - "--certificatesresolvers.stepca.acme.caServer=https://<YOUR-STEP-CA-SERVER>:9000/acme/acme/directory"
      - "--certificatesresolvers.stepca.acme.certificatesDuration=2160"
      - "--certificatesresolvers.stepca.acme.email=YOUR@EMAIL.ADDRESS"
      - "--certificatesresolvers.stepca.acme.httpChallenge.entryPoint=http"
```

Restart Traefik to load this new static configuration.

### Ensure that Traefik Trusts Your Certificate Resolver

Traefik needs access to step-ca's root certificate in order to trust the certificate authority during the ACME protocol exchange. One way to do this is to place the CA root cert on a volume the container has access to and point Traefik at the CA root certificate via the LEGO_CA_CERTIFICATES environment variable on the Traefik container.

In this demo I've been configuring Traefik using docker compose, but for my production Traefik instance I keep my static configuration in `/etc/traefik/traefik.yml` on the Docker host and mount this volume to `/etc/traefik` in the container, so this is a convenient place to also put the CA root certificate. You can do something similar. So, on the Docker host, if you've installed the step client there, you can:

```sh
dockerhost$ cd /etc/traefik
dockerhost$ step ca root root_ca.crt
```

This will save the root certificate in `root_ca.crt` locally.

Then, on the Traefik container (this assumes that you like me are mounting the host's /etc/traefik directory as a volume at /etc/traefik in the Traefik container) set the following environment variable:

```
LEGO_CA_CERTIFICATES: /etc/traefik/root_ca.crt
```

Restart Traefik.

### Enable Automatic Certificate Management for Your Services

Now Traefik should have everything it needs to manage certificates on behalf of the services it is proxying. Here are the traefik-related labels on our example `whoami` service container to let Traefik manage certs.

```
traefik.enable=true
traefik.http.routers.whoami.entryPoints=http,https
traefik.http.routers.whoami.rule=Host("whoami.dockerhost.local")
traefik.http.routers.whoami.tls=true
traefik.http.routers.whoami.tls.certresolver=stepca
```

That last label is the new bit. Now when this container is started up, the Traefik service detects it and determines that it needs a certificate. Traefik then uses the ACME protocol to contact the step-ca certificate authority and ask for a certificate for `whoami.dockerhost.local`. The CA then issues an HTTP challenge back to Traefik in order to prove control of the domain. To meet this challenge, Traefik needs to host a random number given to it by the CA at the URL `http://whoami.dockerhost.local/.well-known/acme-challenge` on port 80. If the CA can reach this URL and verify the number returned is the number provided, it accepts this as proof that you do indeed control the domain for the requested certificate.

Here we can see some of this traffic in the step-ca container log:

```sh
# here is the new certificate order coming in
time="2024-11-05T20:45:26Z" level=info duration=16.087999ms duration-ns=16087999 fields.time="2024-11-05T20:45:26Z" method=POST name=ca nonce=d0NNOTJrSmMxSjlGbzdPODBtSFZPZmY2dVE4Z09sZEE path=/acme/acme/new-order protocol=HTTP/1.1 referer= remote-address=172.17.0.1 request-id=a59884c6-fa8e-4e4e-9947-74c8ac058e6e response="{\"id\":\"3yBYfiFz3YGPBaC1m8ettrWbN23tIPxN\",\"status\":\"pending\",\"expires\":\"2024-11-06T20:45:26Z\",\"identifiers\":[{\"type\":\"dns\",\"value\":\"whoami.dockerhost.local\"}],\"notBefore\":\"2024-11-05T20:44:26Z\",\"notAfter\":\"2024-11-06T20:45:26Z\",\"authorizations\":[\"https://YOUR-STEP-CA-SERVER:9000/acme/acme/authz/5WoJv4DYLvJo5RH2AORzo5QBv6V05nrz\"],\"finalize\":\"https://YOUR-STEP-CA-SERVER:9000/acme/acme/order/3yBYfiFz3YGPBaC1m8ettrWbN23tIPxN/finalize\"}" size=423 status=201 user-agent="containous-traefik/3.1.6 xenolf-acme/4.18.0 (release; linux; amd64)" user-id=

# This is the CA responding with different challenge options
time="2024-11-05T20:45:26Z" level=info duration=4.00311ms duration-ns=4003110 fields.time="2024-11-05T20:45:26Z" method=POST name=ca nonce=U0ExV3o5TU5KUmY4NTdEWjVVNWhmVWlEVXpmZGszbEc path=/acme/acme/authz/5WoJv4DYLvJo5RH2AORzo5QBv6V05nrz protocol=HTTP/1.1 referer= remote-address=172.17.0.1 request-id=1c684657-3523-479a-8fcc-c6f487c86ce7 response="{\"identifier\":{\"type\":\"dns\",\"value\":\"whoami.dockerhost.local\"},\"status\":\"pending\",\"challenges\":[{\"type\":\"dns-01\",\"status\":\"pending\",\"token\":\"FmVIP9MWc8vsu0TUdahVmOtDuOQ2NXHy\",\"url\":\"https://YOUR-STEP-CA-SERVER:9000/acme/acme/challenge/5WoJv4DYLvJo5RH2AORzo5QBv6V05nrz/WZIKzrEYOnuh3nWOAjbqSwHZ4OClHcvC\"},{\"type\":\"http-01\",\"status\":\"pending\",\"token\":\"FmVIP9MWc8vsu0TUdahVmOtDuOQ2NXHy\",\"url\":\"https://YOUR-STEP-CA-SERVER:9000/acme/acme/challenge/5WoJv4DYLvJo5RH2AORzo5QBv6V05nrz/wjUpNeEdMNaXRddjcnAYh0ZInj9G3veT\"},{\"type\":\"tls-alpn-01\",\"status\":\"pending\",\"token\":\"FmVIP9MWc8vsu0TUdahVmOtDuOQ2NXHy\",\"url\":\"https://YOUR-STEP-CA-SERVER:9000/acme/acme/challenge/5WoJv4DYLvJo5RH2AORzo5QBv6V05nrz/78eml5XS3qmffo7EXmgrOel893pE52bK\"}],\"wildcard\":false,\"expires\":\"2024-11-06T20:45:26Z\"}" size=759 status=200 user-agent="containous-traefik/3.1.6 xenolf-acme/4.18.0 (release; linux; amd64)" user-id=

# This is the CA reporting that the HTTP challenge was successful
time="2024-11-05T20:45:26Z" level=info duration=6.848207ms duration-ns=6848207 fields.time="2024-11-05T20:45:26Z" method=POST name=ca nonce=MjBjeUxNRWJ5cDV4RTFJUmp3azJiTmNPNGFRQjhpYlM path=/acme/acme/challenge/5WoJv4DYLvJo5RH2AORzo5QBv6V05nrz/wjUpNeEdMNaXRddjcnAYh0ZInj9G3veT protocol=HTTP/1.1 referer= remote-address=172.17.0.1 request-id=b5147639-768c-461c-b086-14b3ce0bdd34 response="{\"type\":\"http-01\",\"status\":\"valid\",\"token\":\"FmVIP9MWc8vsu0TUdahVmOtDuOQ2NXHy\",\"validated\":\"2024-11-05T20:45:26Z\",\"url\":\"https://YOUR-STEP-CA-SERVER:9000/acme/acme/challenge/5WoJv4DYLvJo5RH2AORzo5QBv6V05nrz/wjUpNeEdMNaXRddjcnAYh0ZInj9G3veT\"}" size=237 status=200 user-agent="containous-traefik/3.1.6 xenolf-acme/4.18.0 (release; linux; amd64)" user-id=

# This is the CA issuing the certificate
time="2024-11-05T20:45:27Z" level=info certificate="XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" duration=3.943992ms duration-ns=3943992 fields.time="2024-11-05T20:45:27Z" issuer="Example CA" method=POST name=ca nonce=S2h4aGxxeHE2MzVjTkRTNnI4VGhTV1NaeFdsUE5yZGw path=/acme/acme/certificate/0GlruhqDVCFxytPLPukee94imRBvPdJQ protocol=HTTP/1.1 provisioner=acme public-key="RSA 4096" referer= remote-address=172.17.0.1 request-id=3282a571-8d8c-402f-9ce5-1065e3b45acc serial=44498717921799979625743144040558764131 size=2063 status=200 subject=whoami.dockerhost.local user-agent="containous-traefik/3.1.6 xenolf-acme/4.18.0 (release; linux; amd64)" user-id= valid-from="2024-11-05T20:44:26Z" valid-to="2024-11-06T20:45:26Z"
```

There's a great in-depth graphic of the full ACME protocol process [at Smallstep's blog](https://smallstep.com/blog/private-acme-server/).

Now, if we visit the service with HTTPS, we can see it has a valid certificate. No more `--insecure` needed!

```
$ curl -v https://whoami.dockerhost.local/
* Host whoami.dockerhost.local:443 was resolved.

[SNIP]

* Server certificate:
*  subject: CN=whoami.dockerhost.local
*  start date: Nov  5 20:44:26 2024 GMT
*  expire date: Nov  6 20:45:26 2024 GMT
*  subjectAltName: host "whoami.dockerhost.local" matched cert's "whoami.dockerhost.local"
*  issuer: O=Sherwood; CN=Sherwood Intermediate CA
*  SSL certificate verify ok.
*   Certificate level 0: Public key type RSA (4096/152 Bits/secBits), signed using ecdsa-with-SHA256
*   Certificate level 1: Public key type EC/prime256v1 (256/128 Bits/secBits), signed using ecdsa-with-SHA256
*   Certificate level 2: Public key type EC/prime256v1 (256/128 Bits/secBits), signed using ecdsa-with-SHA256

[SNIP]

Hostname: e188d01a55f1
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.5
RemoteAddr: 172.17.0.3:47890
GET / HTTP/1.1
Host: whoami.dockerhost.local
User-Agent: curl/8.5.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 192.168.10.50
X-Forwarded-Host: whoami.dockerhost.local
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: c16b32a965e3
X-Real-Ip: 192.168.10.50

* Connection #0 to host whoami.dockerhost.local left intact
```

We can see that the cert is only valid for a day. No worries! Traefik will use ACME to renew the cert every night. Here it is doing just that for my Paperless-NGX service:

```
2024-10-29T03:52:42Z INF Renewing certificate from LE : {Main:paperless.MYHOME.DOMAIN SANs:[]} acmeCA=https://MY.PRODUCTION.STEP.CA:9000/acme/acme/directory providerName=stepca.acme
2024-10-30T03:52:42Z INF Renewing certificate from LE : {Main:paperless.MYHOME.DOMAIN SANs:[]} acmeCA=https://MY.PRODUCTION.STEP.CA:9000/acme/acme/directory providerName=stepca.acme
2024-10-31T03:52:42Z INF Renewing certificate from LE : {Main:paperless.MYHOME.DOMAIN SANs:[]} acmeCA=https://MY.PRODUCTION.STEP.CA:9000/acme/acme/directory providerName=stepca.acme
2024-11-01T03:52:42Z INF Renewing certificate from LE : {Main:paperless.MYHOME.DOMAIN SANs:[]} acmeCA=https://MY.PRODUCTION.STEP.CA:9000/acme/acme/directory providerName=stepca.acme
2024-11-02T03:52:42Z INF Renewing certificate from LE : {Main:paperless.MYHOME.DOMAIN SANs:[]} acmeCA=https://MY.PRODUCTION.STEP.CA:9000/acme/acme/directory providerName=stepca.acme
```

This was a lengthy journey, but one that's worth it. We now can have Docker-based services with friendly DNS names, proxied by Traefik so we can ensure HTTPS traffic to them and not have to publish a lot of ports out of Docker, and we can avoid certificate warnings by issuing and maintaining proper certificates. Do we need all of this in a homelab environment? Of course not, but who said anything about need? This is a great learning experience and does bring a touch of elegance to our homelab setup, don't you think?