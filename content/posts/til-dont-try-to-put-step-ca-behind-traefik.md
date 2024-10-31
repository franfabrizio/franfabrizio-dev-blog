---
title: "TIL: Don't Try to Put step-ca Behind Traefik"
date: 2024-10-28T23:04:01-05:00
draft: false
toc: true
---

Just a quick post to stop others from going down this rabbit hole...

Don't try to put step-ca behind Traefik. I run step-ca as a Docker container, so I thought, why not try to put it behind Traefik with my other services? Here's why not.

Traefik by default terminates SSL at itself and talks plain HTTP to the backend containers. Try that with step-ca and you'll get a `client sent an HTTP request to an HTTPS server` error. step-ca requires HTTPS connections (makes sense, it's a CA after all).

Ok, now you think "I need to do the [Traefik TLS passthrough](https://doc.traefik.io/traefik/routing/routers/#passthrough) thing". But this gets complicated because you need to set up both TCP and HTTP routers. Then, even if you try that you'll get an `Internal Server Error` and the logs will show a TLS handshake error.

The reason for that is that the domain name on the cert needs to match the domain name of the URL, and it won't since behind the scenes docker/traefik is routing with IP addresses.

There's a way to disable TLS cert validation in Traefik (see the `insecureskipverify` option), but that's not a good idea on principle.

You could maybe create a different cert that has the docker IP address, but that's not really intended to be static and kinda works against the whole point of docker and containers and their "ephemeralness". There's probably a solution here but it doesn't feel worth it - really the only thing I gain from it is to not have to specify the port number of the CA in the url, and I'm not often even connecting to the CA as a human (really only to retrieve the roots.pem cert to add it to a trust store somewhere).

Here's a guy that [got something similar to work](https://blog.alexanderhopgood.com/traefik/letsencrypt/2020/07/18/traefik-tls-passthrough), but not with step-ca, if you really want to pursue this further.



