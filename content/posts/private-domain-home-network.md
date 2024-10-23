---
title: "Homelab 101 - Setting Up a Private Domain with Technitium"
date: 2024-10-17T00:12:51-05:00
draft: true
---

## Introduction

Somewhere on your homelab journey you will eventually come to the realization that you would like to have friendly names for all of the things on your home network instead of dealing with IP addresses. There are many ways to scratch that itch, but one of the most common approaches is to run private DNS and DHCP services and point all of your internal devices at them. Here's an overview of the things you'll need to plan and decide in order to get a private domain working on your home network.

## What's a Private Domain?

A private domain is simply a domain name that you'd like to use on your internal network that is not visible from the outside world. This _can_ be an actual honest-to-goodness domain name, but that's not necessary and also problematic if you're planning to use that same domain name for anything public-facing.

The good news is that you can just use whatever domain name you want, including the TLD (the last part of the domain name). If you want your home network to be called `unicorns.donuts`, you can do that. However, there are some pseudo-standards, or at least conventions, that tend to be popular.

Common TLDs for private networks include `.intranet`, `.internal`, `.home`, `.private` or `.lan`. There are a couple I'd recommend not using, as well. `.local`, because that's sort of reserved for something called multicast DNS, or mDNS. In fact, the recommendations I made above come from [Appendix G of the Multicast DNS RFC](https://www.rfc-editor.org/rfc/rfc6762#appendix-G). Another I'd recommend against, even though it's sorta the most official one for this purpose, is `.home.arpa`, which is proposed as a standard in [IETF's RFC 8375](https://datatracker.ietf.org/doc/html/rfc8375). It will work quite well for the purpose, but it's just odd and ugly to me, and is not in common use.

## Providing Private DNS Service

For most home networks, their router (either the one provided by the ISP or something that was purchased like a wifi mesh router system) will serve as their nameserver and the DHCP server (usually also built into the same router) will set clients' nameserver to be the router by default. The router isn't really mapping names to IPs (known as resolving) itself - it's typically just a DNS forwarder. A DNS forwarder forwards queries to full-fledged DNS resolvers, and then it caches the results it gets back for faster lookup next time.

Once you have a private domain name, that needs to change. Now you need a local DNS resolver to resolve those local addresses, because no one upstream is going to know anything about your `kookaburra.home` domain. You need something locally that can help systems map these names to actual network IP addresses. There are many solutions here, ranging from the really simple, like putting an `/etc/hosts` file with all of the local names on each machine that needs them, all the way up to running an actual old school bind9 DNS server.

There are a number of solutions that aim somewhere in the middle of these two ends of the spectrum. One you've probably heard of is Pi-Hole, though you may not realize it can serve this function. Pi-Hole is best known as an ad-blocking solution, but the way it achieves this is useful for our needs here. Pi-hole blocks ads by intercepting DNS queries and cross-referencing lists of known ad-serving domain names. It then blocks those from resolving, so that your local clients cannot retrieve the ads. It forwards non-blocked requests to upstream DNS servers to be resolved. In other words, Pi-hole is a DNS forwarder - at least by default. However, Pi-hole can also be a resolver for local DNS names, meaning that it can map names to IPs for your custom internal domain. If you're already running Pi-hole, see the "Local DNS" section of the settings for more information.

I started my homelab running my own bind9 instance, which appealed to my 1990's sysadmin sensibilities, but I eventually settled on another product which inhabits this middle ground - Technitium. If Pi-hole is an ad-blocker with some DNS functionality attached, then Technitium is a DNS server with some ad-blocking capability attached. Technitium supports a fuller set of DNS functionality than Pi-hole while being slightly less robust (but still plenty useful) on the ad-blocking side. I prefer it for some of that extra DNS functionality - Pi-hole is a DNS forwarder and can be a pseudo-resolver for custom local DNS names, but it's not a real recursive DNS resolver (one which can recursively query DNS servers until it discovers the IP address) without the addition of some additional software like unbound to do the recursive resolving. Technitium is a self-contained package that does all of this, and that's quite useful and handy.

Whatever solution you settle on, make sure your local network contains a DNS server which can respond with the IP addresses for your custom domain devices.

## Managing Dynamic Host Configuration (DHCP)

Once you grow beyond a couple of devices you're going to quickly tire of manually configuring network and nameserver settings on them. You'll need DHCP service to ensure that your devices automatically get the names, IPs, and nameserver settings you want them to get. Here again there are a variety of solutions. The key feature you want is for your DHCP server to update your DNS records when new leases are assigned, so that you can ensure that names map to the right IP addresses and vice versa, even as devices like laptops and phones come and go from your network.

I've run isc-dhcp-server instances in the past to complement the old school bind9 approach, and it does work, but it takes a fair amount of finnicky configuration and upkeep - Ars Technica's Lee Hutchinson [did a nice writeup](https://arstechnica.com/information-technology/2024/02/doing-dns-and-dhcp-for-your-lan-the-old-way-the-way-that-works/#page-4) of this recently. This was the main reason I adopted Technitium - it integrates DHCP with DNS right out of the box, ensuring that the two stay in sync and that your private domain records are always up to date, and provides a nice web UI to manage it all, allowing me to retire both bind9 and isc-dhcp-server (along with pi-hole since Technitium also supports ad-blocking).

The DHCP server can be configured to give out the address of your DNS server as the primary nameserver, meaning that any new devices that join your network can resolve all your local device names and also get ad-blocking automatically.


