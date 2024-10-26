---
title: "Creating a Private Certificate Authority for Your Homelan with step-ca"
date: 2024-10-25T22:55:10-05:00
draft: false
---

## Introduction

In this post, we'll create a private certificate authority (CA) using [Smallstep's step-ca](https://smallstep.com/docs/step-ca/) software. Why do we want to do this? One of the biggest annoyances with running a homelab is that you have to deal with lots of "this site isn't secure" warnings when browsing to all of your internal HTTPS services because you don't have valid certs from a known CA. It would be great if we could issue valid certificates for our internal services that are using our internal domain name that we set up in an earlier post, and that's what we're going to do today!

## Why step-ca?

Probably the first solution that you may have thought of to this problem is [Let's Encrypt](https://letsencrypt.org/). Let's Encrypt is indeed a phenomenally useful service, but it's not quite the right fit for this use case for a couple of reasons. First, Let's Encrypt requires that the DNS entries for your services be public. While we could manage our private domain's DNS entries using a public DNS service (yes, you can put private IPs in a public DNS server, although it's not a great idea), that basically telegraphs our internal network contents to the world, which isn't the best idea. Second, it requires that we open access to the services we want to protect so that Let's Encrypt can do its validation process. For services that you don't ever intend to expose to the internet, that represents an unnecessary vulnerability we'd rather avoid.

So the right solution here is to set up an internal, private certificate authority. There are a number of solutions for this, including just doing it [manually using `openssl`](https://arminreiter.com/2022/01/create-your-own-certificate-authority-ca-using-openssl/). That gets a little finnicky to manage all of the various commands needed, so I prefer to use a purpose-built software solution instead. There are a few out there to choose from. One that I like is [XCA](https://hohnstaedt.de/xca/), and in fact I started out with XCA. I switched to step-ca for one important reason: step-ca is an _online_CA which supports the ACME protocol, the same protocol Let's Encrypt uses. What that means is that you can have your own private Let's Encrypt-esque automated certificate issuer/renewer service for your homelab! If you do a lot of Docker this is great, as you can integrate it with something like a Traefik proxy and have a super slick way to automatically route to your services via a domain name and get and maintain valid certs for them when they spin up, greatly simplifying your Docker environment management. We'll cover ACME usage in a separate post - for today, let's just focus on getting the CA up and running.

## Installing `step-ca` server and `step` client

Smallstep provides [instructions for installing](https://smallstep.com/docs/step-ca/installation/) on a myriad of platforms. 

In production I use the Docker instance, but for this tutorial I've created a test Ubuntu environment and we'll install it using the native packages. I recommend you set it up wherever you want to ultimately run it from, whether that's Docker, a VM, an existing machine. It'll just make it easier to put this into production for you.

So, if you're not setting it up on an Ubuntu host, you don't need to follow my steps - you can click on that link above, install `step-ca` and the `step` client in your desired format, and rejoin me in the "Initializing Your CA" section below. If you want your `step` client to be on some machine other than the `step-ca` server for convenience of managing certs (e.g. your desktop PC), that's fine - but also install it on the server, you'll want it there for some operations below.

Ok, with that said, let's first get the `step-ca` package:
```
fran@step-ca-test:~$ wget https://dl.smallstep.com/certificates/docs-ca-install/latest/step-ca_amd64.deb
--2024-10-26 11:48:12--  https://dl.smallstep.com/certificates/docs-ca-install/latest/step-ca_amd64.deb
Resolving dl.smallstep.com (dl.smallstep.com)... 54.203.228.158, 52.32.4.234, 54.186.175.150
[SNIP]
HTTP request sent, awaiting response... 200 OK
Length: 15258522 (15M) [application/octet-stream]
Saving to: ‘step-ca_amd64.deb’

step-ca_amd64.deb             100%[=================================================>]  14.55M  41.7MB/s    in 0.3s

2024-10-26 11:48:14 (41.7 MB/s) - ‘step-ca_amd64.deb’ saved [15258522/15258522]
```

Then install it:

```
fran@step-ca-test:~$ sudo dpkg -i step-ca_amd64.deb
[sudo] password for fran:
Selecting previously unselected package step-ca.
(Reading database ... 34495 files and directories currently installed.)
Preparing to unpack step-ca_amd64.deb ...
Unpacking step-ca (0.27.5) ...
Setting up step-ca (0.27.5) ...
fran@step-ca-test:~$
```

And now we'll do the same for the `step` client. First download:

```
fran@step-ca-test:~$ wget https://dl.smallstep.com/cli/docs-ca-install/latest/step-cli_amd64.deb
--2024-10-26 11:51:25--  https://dl.smallstep.com/cli/docs-ca-install/latest/step-cli_amd64.deb
Resolving dl.smallstep.com (dl.smallstep.com)... 54.203.228.158, 52.32.4.234, 54.186.175.150
[SNIP]
HTTP request sent, awaiting response... 200 OK
Length: 13592522 (13M) [application/octet-stream]
Saving to: ‘step-cli_amd64.deb’

step-cli_amd64.deb            100%[=================================================>]  12.96M  4.97MB/s    in 2.6s

2024-10-26 11:51:29 (4.97 MB/s) - ‘step-cli_amd64.deb’ saved [13592522/13592522]
```

Then install:

```
fran@step-ca-test:~$ sudo dpkg -i step-cli_amd64.deb
Selecting previously unselected package step-cli.
(Reading database ... 34498 files and directories currently installed.)
Preparing to unpack step-cli_amd64.deb ...
Unpacking step-cli (0.27.5-1) ...
Setting up step-cli (0.27.5-1) ...
update-alternatives: using /usr/bin/step-cli to provide /usr/bin/step (step) in auto mode
fran@step-ca-test:~$
```

## Initializing Your CA

The `step` client comes with a `ca init` command to perform the initial setup of your CA. This should run on the `step-ca` server.

```fran@step-ca-test:~$ step ca init
✔ Deployment Type: Standalone
What would you like to name your new PKI?
✔ (e.g. Smallstep): Unicorn CA
What DNS names or IP addresses will clients use to reach your CA?
✔ (e.g. ca.example.com[,10.1.2.3,etc.]): ca.unicorn.home,192.168.10.126
What IP and port will your new CA bind to? (:443 will bind to 0.0.0.0:443)
✔ (e.g. :443 or 127.0.0.1:443): :443
What would you like to name the CA's first provisioner?
✔ (e.g. you@smallstep.com): fran@franfabrizio.dev
Choose a password for your CA keys and first provisioner.
✔ [leave empty and we'll generate one]: chocolatedonutsareyummy!

Generating root certificate... done!
Generating intermediate certificate... done!

✔ Root certificate: /home/fran/.step/certs/root_ca.crt
✔ Root private key: /home/fran/.step/secrets/root_ca_key
✔ Root fingerprint: bc99c30f52882faaf44475133acd705c03ece9941d23cd569f9362af22d19dcc
✔ Intermediate certificate: /home/fran/.step/certs/intermediate_ca.crt
✔ Intermediate private key: /home/fran/.step/secrets/intermediate_ca_key
✔ Database folder: /home/fran/.step/db
✔ Default configuration: /home/fran/.step/config/defaults.json
✔ Certificate Authority configuration: /home/fran/.step/config/ca.json

Your PKI is ready to go. To generate certificates for individual services see 'step help ca'.

FEEDBACK 😍 🍻
  The step utility is not instrumented for usage statistics. It does not phone
  home. But your feedback is extremely valuable. Any information you can provide
  regarding how you’re using `step` helps. Please send us a sentence or two,
  good or bad at feedback@smallstep.com or join GitHub Discussions
  https://github.com/smallstep/certificates/discussions and our Discord
  https://u.step.sm/discord.
fran@step-ca-test:~$
```

There we go, we've set up a standalone `Unicorn CA` that will respond on port 443 at our private domain name `ca.unicorn.home` or its IP address, and we've configured a first provisioner `fran@franfabrizio.dev`.

**IMPORTANT: Copy your root fingerprint and save it somewhere safe. You'll need it again.**

### Set up DNS and DHCP as Necessary

If you gave your CA a domain name, now's the time to set up the DNS records so that clients can resolve that name to the CA machine. How you do that will depend on how you're providing private DNS service in your environment.

Similarly, the CA needs to have a consistent IP, as it will verify that the IP you're using to contact the CA is one of the IP addresses you provided at initialization time.


### Starting your `step-ca` server

Let's start our server:
```
fran@step-ca-test:~$ step-ca $(step path)/config/ca.json
badger 2024/10/26 12:12:38 INFO: All 0 tables opened in 0s
badger 2024/10/26 12:12:38 INFO: Replaying file id: 0 at offset: 0
badger 2024/10/26 12:12:38 INFO: Replay took: 98.844µs
Please enter the password to decrypt /home/fran/.step/secrets/intermediate_ca_key: chocolatedonutsareyummy!
2024/10/26 12:12:48 Building new tls configuration using step-ca x509 Signer Interface
2024/10/26 12:12:48 Starting Smallstep CA/0.27.5 (linux/amd64)
2024/10/26 12:12:48 Documentation: https://u.step.sm/docs/ca
2024/10/26 12:12:48 Community Discord: https://u.step.sm/discord
2024/10/26 12:12:48 Config file: /home/fran/.step/config/ca.json
2024/10/26 12:12:48 The primary server URL is https://ca.unicorn.home:443
2024/10/26 12:12:48 Root certificates are available at https://ca.unicorn.home:443/roots.pem
2024/10/26 12:12:48 Additional configured hostnames: 192.168.10.126
2024/10/26 12:12:48 X.509 Root Fingerprint: bc99c30f52882faaf44475133acd705c03ece9941d23cd569f9362af22d19dcc
2024/10/26 12:12:48 Serving HTTPS on :443 ...
```

There we go, we're up and running on port 443. One cool thing to note is that step-ca conveniently makes the root certificates available at the url `https://ca.unicorn.home/roots.pem`, which is handy because we'll need those later to tell our clients to trust our new CA.

## Setting up your `step` client

You'll interact with your CA using the `step` CLI client.

### Making `step` Trust Your CA

In this tutorial I'm setting up the `step-ca` server and `step` client on the same system in my user's home directory, so in this case, the step client already has access to the root certificates (which were placed in `/home/fran/.step` above when we initialized the CA) and will trust the CA. However, in many cases you'll want your `step-ca` server to be on a different machine than your `step` client (e.g. Docker server and client on your Windows PC). In those cases, since the client doesn't have local access to the root certificates the client doesn't trust your new CA by default and you need to set that up. 

Here's what it looks like when you try to interact with a CA your client doesn't trust:

```
stepuser@step-ca-test:~$ step ca certificate localhost srv.crt srv.key --ca-url https://ca.unicorn.home/
flag '--root' is required unless the '--token' flag is provided
```

Here, I've tried to create a certificate, but the client says "Hey, you've got to give me a root certificate or some other way to trust this CA".

Handily, `step` provides a `ca bootstrap` command to set up a trust relationship with a CA. Here's one place you'll need that root fingerprint you saved earlier.

```
stepuser@step-ca-test:~$ step ca bootstrap --ca-url https://ca.unicorn.home/ --fingerprint bc99c30f52882faaf44475133acd705c03ece9941d23cd569f9362af22d19dcc
The root certificate has been saved in /home/stepuser/.step/certs/root_ca.crt.
The authority configuration has been saved in /home/stepuser/.step/config/defaults.json.
stepuser@step-ca-test:~$ 
```

### Installing Your CA as a System-Wide Trusted Authority

`step` also comes with a command to install your root CA certificates into your system's trust store. This is helpful so that other utilities besides `step` (like `curl` or `wget`) will trust certificates issued by your CA.

```
stepuser@step-ca-test:~$ step certificate install $(step path)/certs/root_ca.crt
[sudo] password for stepuser:
Certificate /home/stepuser/.step/certs/root_ca.crt has been installed.
X.509v3 Root CA Certificate (ECDSA P-256) [Serial: 2928...9877]
  Subject:     Unicorn CA Root CA
  Issuer:      Unicorn CA Root CA
  Valid from:  2024-10-26T12:05:30Z
          to:  2034-10-24T12:05:30Z
stepuser@step-ca-test:~$
```
The neat thing is that this command will work whether your `step` client is on Windows, MacOS, Linux, or whathaveyou. Smallstep has provided a lot of nice little conveniences.

Now we can check our system's trust store to verify that Unicorn CA is trusted. This varies by operating system, but here's [one way to check on Ubuntu](https://unix.stackexchange.com/questions/97244/list-all-available-ssl-ca-certificates).

```
stepuser@step-ca-test:~$ awk -v cmd='openssl x509 -noout -subject' '/BEGIN/{close(cmd)};{print | cmd}' < /etc/ssl/certs/ca-certificates.crt
subject=CN = ACCVRAIZ1, OU = PKIACCV, O = ACCV, C = ES
subject=C = ES, O = FNMT-RCM, OU = AC RAIZ FNMT-RCM
subject=C = ES, O = FNMT-RCM, OU = Ceres, organizationIdentifier = VATES-Q2826004J, CN = AC RAIZ FNMT-RCM SERVIDORES SEGUROS
subject=serialNumber = G63287510, C = ES, O = ANF Autoridad de Certificacion, OU = ANF CA Raiz, CN = ANF Secure Server Root CA
subject=C = IT, L = Milan, O = Actalis S.p.A./03358520967, CN = Actalis Authentication Root CA
subject=C = US, O = AffirmTrust, CN = AffirmTrust Commercial
subject=C = US, O = AffirmTrust, CN = AffirmTrust Networking
subject=C = US, O = AffirmTrust, CN = AffirmTrust Premium
subject=C = US, O = AffirmTrust, CN = AffirmTrust Premium ECC
subject=C = US, O = Amazon, CN = Amazon Root CA 1
subject=C = US, O = Amazon, CN = Amazon Root CA 2
[SNIP VERY LONG LIST]
subject=O = Unicorn CA, CN = Unicorn CA Root CA
```

The last line shows that Unicorn CA is now a trusted CA by our system!

## Securing Services with Certificates

Now we're ready to use our CA to secure our services with valid certificates. To demonstrate this, let's first create a simple HTTPS server. Here's a Python script to do that:

```python
import http.server
import ssl


def get_ssl_context(certfile, keyfile):
    context = ssl.SSLContext(ssl.PROTOCOL_TLSv1_2)
    context.load_cert_chain(certfile, keyfile)
    context.set_ciphers("@SECLEVEL=1:ALL")
    return context


class MyHandler(http.server.SimpleHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers["Content-Length"])
        post_data = self.rfile.read(content_length)
        print(post_data.decode("utf-8"))


server_address = ("myservice.unicorn.home", 8443)
httpd = http.server.HTTPServer(server_address, MyHandler)

context = get_ssl_context("site.crt", "site.key")
httpd.socket = context.wrap_socket(httpd.socket, server_side=True)

httpd.serve_forever()
```

Here I'm creating a simple HTTPS server at `myservice.unicorn.home`. I've used `openssl` to self-sign a bogus certificate (`site.key` and `site.crt`) to first show what it looks like with an untrusted cert. What Python's `http.server` does by default is turn the current directory into the web root, so let's give it something useful to serve.

```
echo "Hello world!" > index.html
```

After throwing a little entry for `myservice.unicorn.home` in our `/etc/hosts` pointing back at ourselves, we're all set for our little demo.

Now if we run this (`python3 simple-server.py`) and try to visit it with `curl https://myservice.unicorn.home:8443/`, we see that the certificate is not trusted:

```
fran@step-ca-test:~$ curl https://myservice.unicorn.home:8443/
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
fran@step-ca-test:~$
```

It's not happy with our self-signed cert. Now let's issue a proper cert for this service (make sure the step-ca server is still running): 

```
fran@step-ca-test:~$ step ca certificate myservice.unicorn.home site.crt site.key
✔ Provisioner: fran@franfabrizio.dev (JWK) [kid: bSSJYxUyCeo39k99QSaQ4paID0zyo9b2Eb0Rl4P623I]
Please enter the password to decrypt the provisioner key: chocolatedonutsareyummy!
✔ CA: https://ca.unicorn.home
✔ Would you like to overwrite site.crt [y/n]: y
✔ Would you like to overwrite site.key [y/n]: y
✔ Certificate: site.crt
✔ Private Key: site.key
fran@step-ca-test:~$
```

When we restart the python server and revisit it with curl, this time we get:

```
fran@step-ca-test:~$ curl https://myservice.unicorn.home:8443/
Hello world!
fran@step-ca-test:~$
```

Success!

## What's Next?

### Establishing Trust with Clients

You'll want to make sure your internal network trusts your new CA. Just like we did above, you can put the `step` client on systems and install the root CA certificate into the system's trust store with the `step certificate install $(step path)/certs/root_ca.crt` command.

Browsers don't always use the system trust store, so you may need to install the root CA certificate in the browser's own trust store. For example, in Chrome, you can manage certificates in the settings at [[chrome://certificate-manager/]]. Other browsers have similar mechanisms.

Remember, you can easily retrieve the Root CA certificate chain at the URL `https://your-ca-server/roots.pem`, and that's the file you'll import into a  certificate trust store. You may need to restart your browser for it to be recognized.

### [Optional] Setting `step-ca` up as a daemon

If you want step-ca to start automatically and be always running, you have a few options. Probably the easiest is to use the Docker container. Alternately, on Linux, you can [set it up as a daemon](https://smallstep.com/docs/step-ca/certificate-authority-server-production/index.html#running-step-ca-as-a-daemon).

## Wrapup

Now that we have a private CA, we can create certificates for all of our internal services and add our root CA to our internal trust stores and have honest-to-goodness working SSL connections on our homelan.

In my next post, I'll explain how to use `step-ca`'s ACME functionality to integrate with Traefik and automatically manage certificates for your Docker containers.