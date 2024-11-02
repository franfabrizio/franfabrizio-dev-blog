---
title: "Setting Up a Traefik Application Proxy for Docker"
date: 2024-11-02T00:21:40-05:00
draft: true
toc: true
---

## Introduction

In this post we'll be setting up a Traefik service in our Docker environment. Traefik is a reverse proxy/load balancer/edge router designed for cloud-native environments. This is useful for a lot of reasons such as performance and security, but one of the most common reasons to do this in your homelab is so that you don't have to expose a bunch of non-standard ports to reach your containerized web services. With Traefik, all of them can respond on port 80, and Traefik will route to the appropriate one based on the domain name in the URL.

## How Traefik Works with Docker

Traefik itself also runs as a container in your Docker environment, and it does an automatic discovery process to find other containers in the Docker environment. When it finds a container it sets up routing rules for it, either automatically for all containers or on a container-by-container basis, based on how you've configured Traefik. You use Docker labels on the individual containers to configure the setup of each service in Traefik.

## Installing Traefik

Install Traefik using the [official Traefik Docker images](https://hub.docker.com/_/traefik). You need to expose at least two ports, port 80 which is where all of the application traffic will come in for Traefik to route, and port 8080, which is Traefik's own dashboaard. You'll also need to attach the container to a persistent configuration file which holds Traefik's static configuration, by convention `/etc/traefik/traefik.yml`. Here's the Docker command:

```
docker run -d -p 8080:8080 -p 80:80 -v $PWD/traefik.yml:/etc/traefik/traefik.yml traefik:v3.2
```

Once you get the instance up and running, verify that you can reach your Traefik Dashboard. It will look similar to this:

![Dashboard](dashboard.png)







