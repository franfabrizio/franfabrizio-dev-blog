---
title: "Overview of Setting Up a Private Domain for Your Home Network"
date: 2024-10-17T00:12:51-05:00
draft: true
---

## Introduction

Somewhere on your homelab journey you will eventually come to the realization that you would like to have friendly names for all of the things on your home network instead of dealing with IP addresses. There are many ways to scratch that itch, but one of the most common approaches is to run private DNS and DHCP services and point all of your internal devices at them. Here's an overview of the things you'll need to plan and decide in order to get a private domain working on your home network.

## What's a Private Domain?

A private domain is simply a domain name that you'd like to use on your internal network that is not visible from the outside world. This _can_ be an actual honest-to-goodness domain name, but that's not necessary and also problematic if you're planning to use that same domain name for anything public-facing.

The good news is that you can just use whatever domain name you want, including the TLD (the last part of the domain name). If you want your home network to be called `unicorns.donuts`, you can do that. However, there are some pseudo-standards, or at least conventions, that tend to be popular.

Common TLDs for private networks include `.intranet`, `.internal`, `.home`, `.private` or `.lan`. There are a couple I'd recommend not using, as well. `.local`, because that's sort of reserved for something called multicast DNS, or mDNS. In fact, the recommendations I made above come from [Appendix G of the Multicast DNS RFC](https://www.rfc-editor.org/rfc/rfc6762#appendix-G). Another I'd recommend against, even though it's sorta the most official one for this purpose, is `.home.arpa`, which is proposed as a standard in [IETF's RFC 8375](https://datatracker.ietf.org/doc/html/rfc8375). It will work quite well for the purpose, but it's just odd and ugly to me, and is not in common use.

## I've Decided on a Name. What's Next? DNS Service.

For most home networks, their router (either the one provided by the ISP or something that was purchased like a wifi mesh router system) will serve as their local DNS nameserver and the DHCP server (usually also built into the same router) will set clients' nameserver to be the router. The router isn't really mapping names to IPs (known as resolving) itself - it's typically just a DNS forwarder. A DNS forwarder forwards queries to full-fledged DNS resolvers, and then it caches the results it gets back for a period of time which can be as long as the TTL setting for that domain for faster lookup performance next time.

Once you have a local private domain name, that changes. Now you need a local DNS resolver to resolve those local addresses, because no one upstream is going to know anything about your `kookaburra.home` domain. You need something locally that can help systems map these names to actual network IP addresses. There are many solutions here, ranging from the really simple, like putting an `/etc/hosts` file with all of the local names on each machine that needs them, all the way up to running an actual old school bind9 DNS server.
