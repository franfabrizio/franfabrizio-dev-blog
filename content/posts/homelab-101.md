---
title: "Homelab 101 - Getting Started with Your Homelab"
date: 2024-10-21
draft: false
toc: true
---

## Introduction

This blog is a place for me to gather notes about my homelab, and this first post is a guide intended to get folks started on their homelab journey. This post won't tell you how to implement your homelab - it'll simply lay out the big questions and things you need to think about as you get started. It's a collection of hard-earned knowledge and wisdom that I've gathered as I've gone through my own homelab journey over the past few months. I went through countless reddit threads, YouTube videos, and blogs to muddle my way along, and my hope is that I can give the next person a bit of a leg up.

## What's a Homelab?

Glad you asked! In the purest sense of the term, a homelab is a place where you can test out new equipment and technologies and learn more about everything IT from the comfort of your home. However, over time the term "homelab" came to refer to any situation where you're doing more with your home network than the "typical consumer". If you're running your own router or you've installed open source software like OpenWRT on a router, if you're hosting any sort of service out of your home (e.g. personal website), if you like to tinker with Raspberry Pis, if you're running network-wide ad blocking - you fall under the broad umbrella of homelab.

For our purposes, if you want to have more control over what's happening on your home network than just plugging in the standard router that your ISP gives you, then you are a candidate for setting up a homelab!

## Why Should You Trust Me?

Well, you really shouldn't, at least not blindly. I'm not an expert. If this is Homelab 101, I'm maybe enrolled in Homelab 201 or 301 right now. I'm not totally without credentials - I was a sysadmin in charge of a datacenter for a CS department at a university for much of the 2000s so I've put in plenty of hours setting up and maintaining networks, servers, and services. But I've been in IT managerland for the last decade+, and also for a long time it was one of those things where I didn't always want to bring my work home, so I've been knocking off a fair bit of rust as I've gone on this trip.

I've been slowly laying the groundwork for my homelab over the last few years (for instance, when we remodeled our basement a few years back I established a networking closet and ran tons of ethernet from the new spaces back to the closet), but I really got more involved in homelabbing in 2024 when I was switching ISPs and used that as an excuse to rebuild the home network from the ground up. I wanted a place to capture what I'm learning. I don't intend for this to be an authoritative reference - more of a getting started guide, with pointers to the most useful references I've found along the way.

## What are the Key Parts of a Homelab?

I'm sure everyone has a slightly different answer for this, but here's my list of the components you will need to think about to get started with your homelab:

1. "Prosumer" grade or better networking equipment (and a place to keep it) and potentially server equipment.
2. A network topology, typically segmented for isolation.
3. Core services: DHCP, DNS (for ad-blocking and/or private domain), firewalling, VPN, maybe a certificate authority.
4. Some sort of containerization/virtualization environment.
5. Whatever other toys (hardware and software) you want to play with in your homelab.

## Setting Expectations

You should expect to screw everything up the first couple of times, and to redo a lot of your work. This is a feature, not a bug - that's how we learn so much! It's the journey as much as it is the destination.

You should expect to be doing increased tech support for your family, because you're going to mess up the home network at least a few times!

You should expect to spend some money - not necessarily a lot, but making some key investments will make your homelab experience a lot more pleasant and fun. There are ways to be pretty thrifty!

You should expect to have some late nights. Tech work can suck up the hours like nobody's business!

Most of all, you should expect to have fun, learn a lot, and feel proud of what you're accomplishing.

## Homelab 101 - Planning your Core Equipment

You'll need to assemble some equipment for your homelab. This can suck up lots of money but it doesn't have to. Ebay and other marketplaces are a great source of used enterprise gear. You may also have spare gear in your closet that can be repurposed.

Here are the key pieces:

* **Router**. You need something to connect your homelab with the outside world. You _can_ use the router from your ISP, but you'll almost certainly want to run your own. This can be a flashed spare router (e.g. [OpenWRT](https://openwrt.org/)), a computer with multiple ethernet ports you've configured to be a router (e.g. [opnsense](https://opnsense.org/)), a small business or enterprise router from any of a number of vendors (e.g. [Ubiquiti Unifi](https://ui.com/us/en/introduction), [Mikrotik](https://mikrotik.com/), [TP-Link Omada](https://www.tp-link.com/us/omada-sdn/)), or possibly a firewall appliance (e.g. [Firewalla](https://firewalla.com/)). 2.5GB networking is a nice-to-have. Configuring an old desktop can be an attractive option, but purpose-built hardware can be easier to manage and less power hungry as well.
* **Switch(es)**. You'll have multiple wired devices and you'll want to have good switching. Gigabit managed switches are the baseline here. Depending on how interested you are in the networking side of things, it goes up from there in terms of capabilities and features - if you want to go 2.5Gb networking across the board, be prepared to spend some money if so - a decent compromise might be to get gigabit switches with some SFP ports for higher speed connections/interconnects or to use LAG to trunk multiple gigabit ports together. You may also want to look at some of the integrated ecosystems (e.g. Ubiquity/Unifi, TP-Link Omada) to make management of your entire network infrastructure more straightforward. If you plan to power devices like cameras or access points with Power over Ethernet (PoE), make sure you get at least some PoE-capable switching.
* **Wireless Network**. Three basic options here. 1. Use a standard wifi router in access point mode. You probably have something lying in your closet that'll do in a pinch. 2. Use any of the many mesh wifi network solutions on the market. 3. Install traditional wifi access points. Option 1 is cheapest, option 3 is most flexible. If feasible, it's really nice to hardwire your access points back to your switches for better wifi performance.
* **Uninterruptible Power Supply**. You'll at least want one to handle your core networking gear during short outages or voltage fluctuations. Doesn't have to be fancy, but can be if you want it to be. Lots of options from vendors such as [APC](https://www.apc.com/us/en/product-category/88972-uninterruptible-power-supply-ups/), [Cyberpower](https://www.cyberpowersystems.com/products/ups/), and [Eaton](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups.html). Remember that a UPS' VA rating is not the same as the watt load it can handle (which is generally significantly less).
* Optional Items
  * **A rack**. To keep everything neat, a networking rack can be hugely helpful. These come in all shapes and sizes. Two popular form factors for homelabs are rolling racks that you can tuck under your desk, and wallmount racks that you can put in a closet. These can be short-depth racks unless you're planning on going full-on datacenter-style rackmount servers (rare). There are a lot of rack vendors, I have a wallmount 12U 18" depth open frame swing gate rack from [Navepoint](https://navepoint.com/) that has served me quite well, but the right rack for you is going to be entirely dependent on where you need it to go. If you get a rack, don't forget:
    * Racking hardware (screws and cage nuts)
    * Patch panels to terminate any wiring (match it to your cabling - e.g. Cat6 patch panels for Cat6 cabling)
    * Zip ties
    * Shelves for non-rackmount items
    
    One last note here - if you get a rack, there are some great 3D printing sellers on Etsy that make custom rackmount brackets for gear that doesn't come in rackmount form to class up your homelab aesthetics.
  * **Server(s)**. You're probably going to want to mess around with containerization/virtualization of some sort, or maybe set up a NAS system, and you'll need a system or two to host those. A really popular option here is to repurpose an old desktop. Really any computer can work for this to some extent - in a real pinch you can just use your regular desktop, but it's nicer to have a separate system or two for this. Used Dell Optiplex Small Form Factors are really popular for this because [ebay is flooded with them](https://www.ebay.com/b/Dell-Small-Form-Factors/179/bn_25385857) and they have a quite good power usage profile - 9th gen Intel Core CPUs or so are the sweet spot here for price and power efficiency.
  * **Raspberry Pis / IoT Devices**. If you have IoT sorts of projects you'd like to work on.
  * **A watt meter**. This is useful to understand how much power your gear is consuming. [Cheap Amazon ones](https://www.amazon.com/gp/product/B09BQNYMMM/ref=ppx_yo_dt_b_search_asin_image?ie=UTF8&th=1) are fine for basic purposes.
  * **Ethernet cable-making supplies**. I'm constantly making custom-length cabling. You'll want a spool of cable - Cat6 is fine, cost-effective, easy to work with, and reasonably future-friendly, as it can do 10Gb for 55 meters. Stranded copper for patch cables, [solid copper](https://www.amazon.com/gp/product/B07GVP71FL/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&th=1) for anything going in a wall/attic. If it seems too cheap make sure it's not copper-clad aluminum (CCA), ESPECIALLY if you are going to do any Power over Ethernet - CCA cable is not safe for PoE, it generates too much heat. You'll also want a [stripping/crimping tool](https://www.amazon.com/Pass-Thru-Reliable-Klein-Tools-VDV226-110/dp/B076MGPQZQ/ref=sr_1_1?crid=3L56Z17MBNHKZ&dib=eyJ2IjoiMSJ9.sapODUiNpsXTcw1q4kYfYWY0qC-2w8cdyGmpCKqMOwPwBZrYqI0U1ZkYGb56EijUN4cEEuyJ05ZWLTDhCnLD_k-3V5GQCKKyzdpW6HbTJ0uE9SPYYIOSbfTxGPaGSDJTvKSLCXi8fs63oiDC-w6U7CssaNDZd3LWqdLMwC0keCf-HYr7AqGlqQ-lo4_185qCwqrtVS5WX0mkpN3GE43tp-Tx0qLmr4LYBwvLqz7ofLnz5IYDO8ppVJBbXdf0Z5afp0UJST9eZzKRLwvidzPDBk5yVELXW0s8J6rI89ESxdw.hwXcWQ1SWkckhbzrJQ9d4uW7j7Yy_YvhgfGiz5KQcJI&dib_tag=se&keywords=klein%2Bpass%2Bthrough%2Bcrimper&qid=1729826261&sprefix=klein%2Bpass%2Bthrough%2Bcrimper%2Caps%2C110&sr=8-1&th=1) - pass-through style is easier for beginners, [RJ45 connectors](https://www.amazon.com/Cable-Matters-Through-Stranded-Connectors/dp/B07PXMN2VK/ref=sr_1_3?crid=1ZLAGVI3WIJX4&dib=eyJ2IjoiMSJ9.m_yrkiBeoapHweTBxNLu8TR1l26RorqXRsMlpoHlUg72PC_QkxVN-9l1Lugj_0AxbCd3TaQ7CRgv0773b8M5AwVP3pbf6XyXm-6ynnXtpN2ibWTP89O_py0vFD-DOa43Op3-6THB1z60aiFwUXWPp2CUH8e-1FvahkXMwjrZQDRYHQ1m3KmiooLG6cTg0XSJ21xWsT6l5je-XqlAI7XWvc4LkqqD6OimihnSPzEtOpg.OwXkZQwIcm_BNOc8jeIZ2P07i8mbCmUSEmMySTuyYBA&dib_tag=se&keywords=klein%2Bcat%2B6%2Bpass%2Bthrough%2Brj45%2Bconnectors&qid=1729826331&sprefix=klein%2Brj45%2Bpass%2Bthrough%2B%2Caps%2C111&sr=8-3&th=1), [keystone jacks](https://www.amazon.com/Cable-Matters-25-Pack-Keystone-Punch-Down/dp/B004D5PFGW/ref=sr_1_3?crid=3IHLU5UVK9DCB&dib=eyJ2IjoiMSJ9.pQNfiVaDNTINVvpW5Mz1NKB74xOB3yZF4YBmtjk1WiwfHaa8mrMzTwStbEaRKTOw9qDUzOcl0c5Lewmmur8YdtOLCu2Xz8cwN5yrEY6aj3XNVw2eFcNP0am4acbBT2ZNwsuueOX95n-Lnz29hr9pfjXs70gdF0TK10ylEJ5WFYA8R2OZa_0ZPPbbtPz4-qSODy1ha6HIFWmgVXgW1GpBLsVeGHcg1khuTlDMO4uf-aA.wSJc2e8XwuOyDxbgeNxS2DOjYRv8_x9h7sWII4hpuRE&dib_tag=se&keywords=cat6+keystone+jacks&qid=1729826379&sprefix=cat6+keystone+jacks%2Caps%2C111&sr=8-3) and a [punch-down tool](https://www.amazon.com/Cable-Matters-Punch-Down-Blade/dp/B0072K1QHM/ref=sr_1_1?crid=LP039AJN43O4&dib=eyJ2IjoiMSJ9.iHMjDkRNyG7nXJ0_qWsCCEVnrmcgrmOFp8NZXI-0pcnQ_rJIyK6MnSoOAwnSgtBGpPX4_LIiUShFjA-JVNn1uui8xKl6Vd1RHhcg3Bn8vHsX8OUWFQD8O9ntMrVIbLnGkwUdoGhzeiVgmpWZ_4eOGWfbMi4JGMs1-GYslbQPPKf_GXhaY9ZUDksGbr5c-uiCYmj5lSDkFwTUueX_UDaf_d7B9mQp9EMnsGCxLDFBX1w.jjGmQPv4sbTPySsyjvTGUC4w_yj5CXUnY9V0tYqPZHQ&dib_tag=se&keywords=cat6+keystone+jacks+punch+down+tool&qid=1729826418&sprefix=cat6+keystone+jacks+punch+down+too%2Caps%2C108&sr=8-1) if you're terminating cables in a rack. These investments have paid off for me time and again over the years.
  * **A label maker**. Well-labeled equipment is one of life's simple pleasures. [Brother P-Touch](https://www.staples.com/brother-p-touch-pt-h110-portable-label-maker-pth110/product_2290773) is the classic.
  * **A cheap, small monitor and keyboard**. For directly plugging into your docker host or other machines when you screw up their networking. I just use whatever I have around the house, but the rise of these [cheap portable USB-powered monitors](https://www.amazon.com/MNN-Portable-15-6inch-Ultra-Slim-Speakers/dp/B0B9NNWXVP/ref=sr_1_3?dib=eyJ2IjoiMSJ9.zYoSG_Hk4aWDz_0ZleyeXyUZyqEV-85BWRaCG5YwQwFH8Ck2nLULHsxc19ZD52T48B-SYv_aUS6PjXPBZNUkyU0EjtQTfHbvDKWxSz0Oagksv0b-H3f9J6IhMISobMSvgOwWCdLdsGix2LDvHsGnRLeMwFBMB47ElLylSHtXdMY540h1D-wMZJY6gADTD5H77EEVk24X6xtYyWujPePdVPj22Q_iMeob156yZsVuETI.-lJe5wTCGD7wfCoIuiWMno6UxCSCiB9BcuFAuh90g1Y&dib_tag=se&keywords=USB%2BPowered%2BMonitor&qid=1729826604&sr=8-3&th=1) has been tempting me.
  * **USB ethernet adapters**. For doing troubleshooting in the networking closet from your laptop that no longer comes with an ethernet port. I keep [USB-A](https://www.amazon.com/USB-Ethernet-Adapter-Gigabit-Switch/dp/B09GRL3VCN?source=ps-sl-shoppingads-lpcontext&ref_=fplfs&smid=ATVPDKIKX0DER&th=1) and [USB-C](https://www.amazon.com/dp/B08HQBC678?ref=ppx_yo2ov_dt_b_fed_asin_title) variants on-hand.

Make sure to consider your location when planning out your equipment. Think about things like noise levels (in your home office? Consider fanless switching equipment), power consumption (do you live in a high-energy-cost area? Do you have enough circuits in the room to handle your expected load?), tidyness (does this need to tuck away during the work day?).

## Homelab 101 - Planning Your Network

Now that you've got your equipment, you need to think about your network. You can just have one big LAN for your homelab, but this is missing out on a whole lot of the fun and learning. More typically, you'll want to segment your network based on category of devices in order to provide isolation of types of traffic and a better security posture. For example, here's a common model for network segmentation:

* Admin/Management - limited access for privileged people/systems to manage the network infra
* Trusted devices - things like your home PCs and tablets
* IoT/untrusted devices - devices that phone home a lot, devices that you have to control from the cloud from vendors that you don't trust to make secure products
* Guests - the wifi you give visitors so they can get to the Internet, isolated from your internal network
* DMZ - if you are hosting Internet-facing services, to isolate those from the internal network

These can be implemented as actual separate networks, but much more commonly VLANs are used (which is why you want managed switches above).

Other network segments you might want to consider:

* No internet access - for things like security cameras, you may want to put them on an internal-only network with no Internet access
* Lab/experimental - Maybe you don't want the family mad at you every time one of your experiments goes sideways?
* Work - You want to isolate your work-from-home setup because you don't trust your employer to not spy on you

Really, you are limited only by your imagination, but I would encourage you to start simply. The more you have, the more access rules you need to manage and it can become a lot, so wait until you've got the basics down before making it more complicated than it needs to be. You can always divide further in the future.

## Homelab 101 - Core Services

The big one here is DHCP. Normally your ISP's router has an integrated DHCP server and hands out IP addresses to devices as needed. You'll need to provide some mechanism for that. Most routers have the ability to serve as DHCP servers, so depending on what you're using for your router, that might be sufficient. There are reasons you want a dedicated DHCP server, though. One big one is if you are also choosing to have a private domain name (see the next paragraph) - then you ideally want some DHCP-DNS integration. You can run a standalone DHCP server (e.g. [isc-kea](https://www.isc.org/kea/)), but some products also have integrated DHCP servers (e.g. [Pi-Hole](https://pi-hole.net/), [Technitium](https://technitium.com/)) that can work well.

If you want a private domain name (something like `smithfamily.lan`), which I highly recommend, you'll also need to provide DNS service to resolve those private names to IP addresses. This is super handy for giving your devices friendly names like `dockerhost.smithfamily.lan` instead of having to remember that your Docker host is at 192.168.20.55. Again here you can run standalone like [bind9](https://www.isc.org/bind/), or you can use an integrated product like Technitium. This is also where having good integration with DHCP is helpful, so that when new devices appear on the network, they not only get an IP address from the DHCP server, but the DHCP server also auto-manages forward and reverse pointer records in your DNS system for the devices. Then, when your phone pops on the network, you can have it get an IP address and a DNS entry for `mypixel.smithfamily.lan` or whatnot.

You'll need some accommodation for firewall management, so that you can protect your network from external threats, isolate VLANs, and allow traffic to pass across VLANs or to/from the WAN as needed. Again, routers usually have some firewalling functionality built in, but you might want a standalone firewall for more control. Opnsense is a popular open source software firewall that you can set up on a PC, or you can buy a hardware firewall like Fortinet or Firewalla. My general advice is that unless you're specifically interested in firewall management or super-fine-grained control over your network, start with whatever your router provides until it proves insufficient.

If you would like to access your homelab from the outside world, you'll need some way to get in, typically a VPN. This is yet another area where your router may have something built in that you can leverage like [OpenVPN](https://openvpn.net/). If not, you can set up a dedicated VPN service in a container or small PC or even a Raspberry Pi, or maybe you have a NAS appliance like a [Synology](https://www.synology.com/en-us) or [Qnap](qnap.com) with a VPN server built in.

Finally, are you sick of dealing with the invalid certificate warning screens when getting to your homelab web UIs? You can optionally create your own certificate authority to issue valid certificates for your internal services (especially nice when combined with private DNS service). You can do this [manually with openssl](https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/), or there are some decent software packages to do this, like [XCA](https://hohnstaedt.de/xca/) or [step-ca](https://smallstep.com/docs/step-ca/).

## Homelab 101 - Containerization / Virtualization

At some point in your homelab journey (usually pretty early), you're going to want containerized applications or virtual machines. Maybe you're even doing a homelab _specifically_ to learn about technologies like [Kubernetes](https://kubernetes.io/). To do this, you'll need at least one host machine, multiple if you plan to look at clustering solutions. This is where cheap used enterprise PCs off of Ebay could be your friend. As far as the software side, [Docker](https://www.docker.com/) is the king of containerization tech, while [Proxmox](https://www.proxmox.com/en/) is super popular in the homelab space for virtualization. You'll also come across things like [LXD](https://canonical.com/lxd) and of course [VMware](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion) in this space.

One caution - it's tempting to run core services like DNS and DHCP as containers. This can work fine (and I do it), but can also lead to chicken-and-egg headaches when the container host crashes or you suffer a power outage. Then the rest of your network can start to suffer as machines can't come back online or do domain lookups easily. You might prefer to run those services outside of a containerization environment for that reason. (The same is essentially true of virtual machines). But containerization is fantastic for tons of other things - I run services like my certificate authority and Paperless-NGX for my paperless home office as containers.

## Homelab 101 - The Rest

At this point we've covered the basics. Now it's time to ask yourself what specific plans you have for your homelab. If you want to set up a nice surveillance system around your property, you might need a dedicated video recording system. If you're planning to serve up a lot of media with something like Jellyfin or Plex, figure out if you have the right host and storage environment for that. If you're going to dive deep into Raspberry Pi, do you have all of the Pis and accessories you'll need? Also consider your need for ancillary tech like 3D printers.

Having spent some time thinking through these various aspects of your homelab, you're better prepared to start planning your implementation, know what to look for on ebay or in your closet, and have a better idea of what software packages to start researching. I hope this primer was helpful in getting you started on your journey.

**Good luck, learn a lot, and most importantly, have fun!**
