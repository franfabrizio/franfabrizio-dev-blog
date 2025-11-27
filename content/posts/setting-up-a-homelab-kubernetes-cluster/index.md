---
title: "Setting Up a Homelab Kubernetes Cluster"
date: 2025-06-12T16:12:26-05:00
draft: true
toc: true
---

## Introduction

This post details how I set up a 2-node Kubernetes cluster in my homelab. Why? Mostly just for learning. We're migrating some workloads to Kubernetes (k8s) at work and I wanted to get more hands-on experience so that I can better understand the technology. But also, because I can!

## Use Case

My goal was to create a multi-node Kubernetes cluster that would be close enough to the "real thing" to teach me some skills that would be transferrable to the production environment we're creating at work.

## Prequisites

### Hardware

I decided to buy (from ebay) two Dell Optiplex 3070 Small Form Factor PCs with i5-9500 CPUs for this cluster. I chose this particular hardware because they are small-ish, cheap, and pretty power efficient for a rather decent 6-core processor with virtualization support. They live in a different room than my office so size and sound aren't so big of factors that I'd need to consider a mini PC, an Intel NUC, or even Raspberry Pi. 

There are lots of other ways to set up a k8s cluster at home, including running multiple nodes on an existing machine using VMs, but I want to eventually explore k8s' high availability features and I wanted to avoid disruption to the rest of my homelab so I preferred two distinct pieces of hardware.

### Operating System

I installed Ubuntu 24.04 server on these machines. Kubernetes will run on many flavors of Linux as well as MacOS, Windows and even ARM devices like Pis, but Linux is its most natural environment. I chose Ubuntu because it's the distro I am most familiar with and which we use at work, but you have a lot of flexibility to choose what's right for you. Thinking ahead, if you think you may want to use a GUI tool to manage your cluster, you might want to make at least one of the instances a desktop Linux variant instead. 

## Choose a Kubernetes Flavor

The first thing I learned is there are a lot of ways to get from zero to k8s cluster. For homelab use, the three I came across most often were Minikube, k3s, and MicroK8s.

* Minikube - This is great for getting first exposure to Kubernetes, but it's limited to creating a 1-node cluster, so it didn't fit my use case.
* k3s - This really focuses on being a lightweight option. It uses very little resources and runs well on tiny hardware like Raspberry Pis. I absolutely could have used it for my use case.
* MicroK8s - This is the one that's closest to a full Kubernetes setup, which was attractive to me. Also, it's provided by Canonical, and this would be running on Ubuntu machines, so I was confident I could find good documentation. It's a bit more full featured and can do more out of the box with less customization required. It's snap-based, which I know some people dislike, so that might be a reason to use another alternative. This is the Kubernetes flavor that I ultimately chose.

One caution: I'd advise against running a Kubernetes cluster using Docker containers if you can avoid it, as the two environments don't always play well together. If you really need to do this, use a flavor that's specifically designed for it, like Kind (Kubernetes IN Docker) or k3d (a wrapper to k3s designed for Docker).

