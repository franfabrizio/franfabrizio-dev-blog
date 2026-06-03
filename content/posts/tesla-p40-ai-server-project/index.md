---
title: "When Cheap VRAM Gets Complicated: My Tesla P40 AI Server Project Part 1"
date: 2026-06-02T15:48:41-05:00
draft: true
toc: true
---

I believe that the future of AI is local.

Some of that is practical: I don't want every question, experiment, and coding session routed through someone's cloud service. Some of that is financial: I don't think the cloud-based AI business models are friendly to individuals or sustainable more generally. And a lot of it is ethical: I don't believe that AI, and our interaction with it, is something that should be centralized, scaled and controlled by IT megacorps. I want AI that I control and that puts my interests front and center.

That's how I ended up spending the last two months working to create a viable local AI inference server.

I started with my trusty NVIDIA RTX 3060 Ti on my everyday PC, but it did not take long to understand just how limiting 8GB of VRAM could be for local LLM inference. Although I could do some interesting things with casual chat and batch processed tasks, anything involving more complex reasoning - and especially agentic coding assistance - just wasn't able to meet my needs and expectations with 8GB VRAM.

I was able to make a tactical upgrade to my experimental system by trading in my 3060 Ti for a 5060 Ti 16GB card, doubling my VRAM at a modest cost while also improving my gaming setup, which is what this machine is normally used for when not doing AI experimentation. That helped a ton, to be honest -- the combination of double the VRAM and two generations newer tensor cores really improved my AI prospects. I can now run agentic coding workflows on models like Qwen3.6-35B-A3B with 256k context completely reasonably now using some aggressive quantization like Unsloth's UD-IQ3_S.

The problem is that this system is my daily driver. I really wanted an always-on AI server waiting in the background whenever I or the rest of my family members needed it. This would require building a separate server.

My goal shifted to seeing how cheaply I could build a competent, always-on AI inference server. Unfortunately, I picked the absolute worst time for this experiment. Thanks to the AI boom, pretty much everything in computer parts, new or used, has seen prices skyrocket. I would need to get very creative to do this in a financially responsible way.

Getting a high-end consumer GPU with enough VRAM was out of the question, with even 4-year-old used 3090 24GB VRAM cards going for $1,200+. I could get another 5060 Ti 16GB card in the $550 range, but I was hoping to do better than that both on price and VRAM size. AMD's GPUs can be a bit cheaper for a bit more VRAM, but leaving the NVIDIA ecosystem introduces additional headaches and compatibility hiccups for not that much in cost savings, so I wasn't thrilled with my options there, either.

Another route to take would be to get a large unified memory system such as the Apple Mac Mini/Studio or AMD Strix Halo. Unified memory is where the CPU and GPU share memory. It's faster than DDR and slower than GDDR, but you can get systems with 128GB of unified memory or even more, meaning you can run large models, even if they're slower than they would be on GPUs. The problem is that these systems aren't cheap either. You're approaching $2,000 for interesting amounts of memory, these use frameworks other than CUDA, which may introduce headaches I don't want.

I want to do better (meaning cheaper) than this - much better. How far can I push the budget down?

## Teaching an Old Dog New Tricks

Enter the NVIDIA Tesla P40. The P40 is a datacenter-grade GPU that was released in 2016, retailing for $5-6k. Back then it was used for training models for tasks like image classification and object detection. The current AI boom was still years off in the future.

The P40 is a PCIe 3.0 24GB GDDR5 VRAM card with 3,840 Pascal-era CUDA cores, no Tensor cores, and 346 GB/sec memory bandwidth. Compared to today's consumer flagship 5090 with its nearly 22k CUDA cores, 680 5th gen Tensor cores, 32GB of GDDR7 VRAM, and almost 1.8 TB/sec memory bandwidth, the P40 is a dinosaur.

The P40 has exactly two things going for it: the NVIDIA CUDA ecosystem, and more importantly, they are _dirt cheap_ on the secondary market. You can pick up a P40 for about $250 (which is actually more expensive than it was a year or two ago, thanks again AI boom!), whereas you'd be lucky to find a 5090 for less than $4,000. You can literally get 384GB VRAM worth of CUDA-compatible GPU for less than the cost of a single 32GB 5090! That VRAM-per-dollar math is hard to ignore.

Because of this, the P40 has become a favorite GPU of hobbyists looking to build AI inference servers capable of running large models affordably. Many an enthusiast has picked up one or more of these and paired them with old server- or workstation-class hardware to create a surprisingly competent AI inference machine for well under $1000. Now we're talking my language!

## Let's Temper Expectations

Now, it's not all sunshine and roses. At a fairly modest 346GB/sec memory bandwidth, there will be hard limits to how fast this setup can generate tokens. Then there's software compatibility to consider. Yes, these are CUDA cards, but Pascal is a very old GPU architecture at this point. NVIDIA announced that the 580 driver series will be the last to support Pascal-era cards, so the latest drivers will no longer work. For now it's completely possible to get a working software stack together with relatively minimal fuss, but there's no guarantee that will remain the story in the future.

And there are hardware considerations as well. The P40 is a passively cooled card designed to be put in rackmount server chassis with high-pressure front-to-back airflow, so if you're not using that sort of chassis, you have to solve the airflow issue on your own - you can't just put them in a tower case and call it a day. They're 250W TDP power hogs too, so you'll need to ensure adequate power. And, as a datacenter card, it takes EPS connectors for power, not PCIe, so you'll need an adapter for that as well. Each card will require 2x PCIe connectors to adapt to a single EPS connector, so you'll need to figure out how to provide 4 PCIe connectors from the host system. Oh, and no display outputs, so you'll need a different solution for bootstrapping the machine.

Building cheap P40-based inference servers is fun, but go in with your eyes wide open.

## The P40s Needed a Home

Forging ahead, I procured two P40s from ebay for $250 each. Now I needed to turn my attention to a system to house them.

Basically, you need a housing that has 1. ample quality power, 2. room for beefy cards, and 3. is designed to run and cool enterprise class hardware.

I knew I didn't want to use a rackmount server chassis, even though that would be by far the easiest, being the P40's native habitat. I simply don't have a good place to plop a rackmount server, and having run datacenters in my career, I was also hoping to do a little better with noise than the typical rack server.

The next best thing would be a pro-level workstation. These generally are huge and roomy and are built with quality components that hold up under stress. In doing my research, I kept coming across folks who had success with Dell and HP workstations. Specifically, AI kept recommending the Dell Precision T7820 workstation. In single-CPU configurations its specs noted that it came with a 950W power supply, supported two 300W GPUs, and had two PCIe x16 slots for the GPUs and lots of room in the chassis.

The CPU, storage and RAM aren't the critical components here, as inference is GPU bound. Anything halfway decent for CPU would do the trick, enough storage for a few models, and 32GB RAM is sufficient. On eBay I found a T7820 with 32GB of DDR4, 500GB SSD, and a Xeon Silver 4112 processor for $350. It's a bit of a skimpy processor but probably fine for this task. It even had an old Radeon WX3100 GPU in it so I had easy video during setup.

For cooling, I am fortunate to have a 3D printer so I printed out some shrouds designed to attach two high-speed 40mm fans onto the rear of each card to push air through, and picked up a 5-pack of Arctic 15K RPM 40mm x 28mm fans, as well as a cheapo PWM hub to power the four fans.

For power to the card, I got two adapters which convert 2x PCIe into 1 EPS 8-pin power connector. This is crucial; PCIe and EPS are both 8-pin connectors, but they have power and ground reversed. They _shouldn't_ physically fit in each other's ports, but poorly made adapters sometimes don't adhere to standards, and there are stories of fried P40s.

Also be careful to pay attention to proprietary manufacturer parts. Dell LOVES to do this. Dell's PSUs have no cables whatsoever - they just distribute power directly to the motherboard and to peripherals through a power distribution board. I needed some cables to convert the proprietary 10 pin connector for the 2nd (unused) CPU into 2x PCIe connectors. That would power one card, while the included 2x PCIe connectors in the T7820 would power the other. Finally, even Dell's fan headers are proprietary 5-pin, so I needed a cable to convert from a standard header to the Dell header for the PWM hub.

Total BOM:

- 2x P40s - $500
- Dell T7820 w/ 32GB RAM - $350
- Fans - $25
- Shrouds - 3D Printed
- Cables - $50

I came in at $925 all-in for a 48GB VRAM local AI inference server. Not too bad.

## It Looked Good on Paper

The machine shipped with Windows, but I intended to run Linux on this system so first thing I did was to install Ubuntu 26.04 before installing either P40. I wanted to make sure I had a stable Linux baseline to work from. That part went smoothly.

The first sign of trouble was when I added in one of the P40s. Although the system did detect the P40 on the PCIe bus, what followed was a long weekend full of debugging and more reboots than I can count. I'll save the gory details for a dedicated post, but among the issues I encountered were:

- BIOS confusion about which GPU was the display GPU
- Inability of the T7820 to allocate the P40's massive 32GB BAR (Base Address Register) request despite "Memory I/O Above 4G" being enabled in the BIOS.
- Inability of MANY versions of the NVIDIA driver to initialize the card - as soon as it would try the card would drop off the PCIe bus
- Deep troubleshooting on power delivery (multimeters, external power supplies, etc...)
- Innumerable combinations of kernel boot flags to try to overcome the T7820's PCIe limitations
- Repeated attempts on Ubuntu 26.04, 24.04, 22.04 and Windows 11

What looked great on paper was a massive headache in practice. AI wasn't much help either. It was a decent troubleshooting partner, but ultimately became convinced I had bad P40 hardware. However, I dropped the cards one at a time into my Windows 11 gaming machine and proved that they could be managed by NVIDIA drivers there, so I was pretty sure I had solid GPUs.

## Lessons Learned and Next Steps

Trying to build around the T7820 wasn't a crazy idea. I thought I was making a good choice by going with a workstation-class platform from an enterprise vendor. A roomy workstation chassis, a beefy PSU, enterprise reliability and documented support for high-power GPUs felt like a recipe for success.

In reality, the Dell walled garden was actively working against me. The proprietary BIOS and hardware were limiting my options in ways that were frustrating and counterproductive. In the end, I was never able to get the T7820 to initialize a P40 on its PCIe bus, and it increasingly started to feel like this was a Dell-specific problem. For folks with more patience than me that want to try to get this to work, I have detailed all of the various roadblocks and workarounds I tried in a separate post, XXXX, but for me, it was time to move on.

Because the cards worked ok in my AMD AM4-based gaming machine, I stopped trying to fight the Dell and hatched an alternative plan to build a commodity system around the AM4 platform. Running two datacenter-class GPUs on a consumer desktop platform seemed counterintuitive, but after my experience on the Dell workstation I felt I had nothing to lose. With AM4 being the last-gen AMD platform parts would come relatively cheaply, plus since I already had built around AM4 at home I likely had parts I could scavenge as well, which would also help keep costs down.

In my next post, XXX, I'll walk through how I built out that platform and how it worked out.

