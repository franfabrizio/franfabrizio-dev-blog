---
title: "Adventures in Local LLMs Part 2: The Hard Truth about Hardware"
date: 2026-04-28T10:02:18-05:00
draft: false
toc: true
---

## Introduction

After deciding that local LLMs were worth exploring, the next obvious question was: *can I even run these things with the hardware I already have?*

## My Baseline

Before going any further, here’s what I’m working with:

* Ryzen 5900X
* 32GB system RAM
* NVIDIA 3060ti 8GB VRAM GPU
* Windows 11

Nothing exotic. A solid build from a few years ago. Importantly, this is not a purpose-built AI box. It’s my daily driver that I’m trying to repurpose into a usable AI server.

## Let's Define Usable

The metric you'll hear about constantly in this space is tokens/second. Specific models on specific hardware setups will produce a certain number of tokens (roughly words, though not exactly) per second i.e. how fast the AI can "talk" to you. This is of course very important for usability when it comes to AI systems. Ideally, they are coming at you faster than you can read. We can deal with some slowness, but too slow and it quickly becomes very annoying to interact with. So, tokens/sec - our goal is as close to or above our reading speed.

The metric that doesn't get nearly enough attention in my opinion is Time-to-First-Token (TTFT). For local LLMs, this is critically important. Cloud LLMs respond more or less immediately in most cases. For a local LLM, there can be significant lag to TTFT, especially if the model needs to be loaded into memory first. Depending on the size of the model and your hardware, 30 seconds, 60 seconds, or more is fairly common. That's a HUGE usability hit and makes interactive iteration with AI extremely painful.

So, we want to minimize TTFT as much as we can for interactive use cases. For batch/background use cases we don't care so much about TTFT.

## The First Misconception: “I Have Plenty of RAM So I Can Run Large Models, Just Slowly”

One of the first things you'll learn about running LLMs is that to do their work they'll fill your GPU's VRAM first, and if necessary they'll spill over into system RAM. GPU VRAM is very fast compared to system RAM, so there's a performance penalty associated with this spillover effect. I knew that going in, but what I didn't expect was how severe the usability hit would be.

In the world of local LLMs there's a constant tension between model size and VRAM availability. You really want to fit the whole model into VRAM, but you also really want an effective model, and effectiveness usually correlates with model size. At my 8GB VRAM limit, I'm severely limited in the types of models that will fully fit in VRAM. Models are measured by how many billions of parameters they contain - you'll see models labeled 1b, 4b, 9b, 27b, 55b, etc... A good rule of thumb is that for every billion parameters of an uncompressed model roughly 1GB of VRAM is required, though compression ("quantization" in LLM parlance) can reduce that significantly at the cost of model response quality. So, for my 8GB VRAM card, I'm looking at uncompressed models of 8b parameters or less. That's fine for some tasks - quick chat, simple questions - but it's a total no-go for anything requiring complex reasoning, like coding assistance or even just answering a more sophisticated question, or holding a long conversation.

So, for several of my use cases, it appears I will need to spillover into RAM to get anything approaching effective. Sounds fine in theory, I'll just need a little patience, right?

In practice, what I’ve found is:

* **Fits in VRAM → usable**
* **Doesn’t fit in VRAM → frustrating**

Coming back to that all-important TTFT metric, I've seen TTFT go from ~ 3 seconds for a model that is fully resident in VRAM to 20+ seconds once spillover into RAM occurred. The effect is not at all subtle.

For deep research questions where I'm asking my question and walking away to let it think, alright, I can be patient. But for daily chat, coding assistance, and similar tasks - I found if there was spillover into RAM, it was incredibly frustrating to use.

## Why VRAM Matters So Much

LLMs are large, and all of that data needs to be stored somewhere that the GPU can get to fast.

VRAM is:

* **High bandwidth**
* **Low latency**
* **Directly accessible to the GPU**

System RAM is:

* Slower
* Shared with everything else
* Accessed over PCIe (which becomes a bottleneck)

So even though I technically *can* run larger models by spilling into system RAM, the experience degrades quickly. VRAM is the constraint that dominates everything else.

But getting everything within VRAM is no easy task. For one thing, just because your graphics card has 8GB of VRAM, that doesn't mean you can actually use all of it.

### You Don't Have as Much VRAM as You Think

On a daily driver like my system, lots of things want at that VRAM:

* Are you looking at a monitor right now? Those pixels need to live in VRAM.
* Are you running any hardware-accelerated apps? Those tasks need to live in VRAM. Chrome loves to eat this up, for example. And there are surprises here - Slack. Joplin (my notetaking app). Why? Because they're Electron apps - basically little mini-Chromes. And if you're actually gaming - well, forget about it.
* The OS maintains buffers here.

All told, on my dual-monitor Windows 11 system, even with hardware acceleration turned off in Chrome and my electron apps closed, something like 3GB of VRAM is reserved, leaving me about 5GB for AI. Oof.

### Model Size vs Reality

Just looking at the model size is also misleading. What needs to fit into VRAM includes:

* the model weights (what's usually reported as the model size)
* the KV (key-value) cache (affected by context window size)
* some scratch space during runtime. (general LLM overhead)

The KV cache in particular can make a huge impact on what can fit in VRAM. Essentially, the KV cache is a cache of your prompt and response history - effectively the model's memory, or "context", to introduce another LLM term. As the context grows, the KV cache size grows very rapidly, due to how things are represented under the hood. A 4K context -> 16K context is not a small, linear jump in VRAM usage - it's massive. This is because the KV cache is not simply a text record of your prompts and the model's responses - it's the model's internal representation of the conversation, expanded over every layer of the model.

Roughly speaking, considering how much VRAM I actually have to work with on my 8GB card (very roughly):

* <=4B models → comfortable with reasonably small context window sizes (smaller models generally tend to have smaller max context sizes anyhow)
* ~4–14B → viable with patience and the right use case, but expect significant spillover penalties
* 20B+ → not happening locally in any practical sense unless you really can wait a very long time

This is where expectations start colliding with reality. The models that benchmark closest to top-tier cloud models are generally *not* the ones that run well on consumer hardware (though there is hope for the future as this is beginning to change somewhat).

Obviously, the more VRAM you have, the better the experience you can deliver, but just be aware that even at the high end of consumer GPUs (16GB or if you're really lucky, 24GB) there are still going to be plenty of compromises to consider.

### A side note about Apple RAM

One interesting edge case in this local LLM world is Apple's "Unified RAM" architecture, where the Apple Silicon (M1-M5) Mac systems have a single type of RAM shared by the system and graphics processor. Unified RAM is faster than regular RAM but slower than VRAM, and there's often a LOT of it - 64GB or even 128GB on higher end systems. Because of these properties, you can run very large models on Apple hardware with a comparatively modest spillover effect. This is why Macs are so popular right now for homelab AI. But make no mistake, Unified RAM is not in the same league as VRAM. It *will* be significantly slower head-to-head with a VRAM-based model.

## The Second Misconception: “GPU Power Matters Most”

Coming from a gaming perspective, I instinctively think in terms of:

* cores
* clock speed
* raw performance

So my assumption was that current-gen 5000 series NVIDIA cards were going to be far superior to previous generations like my existing 3060ti. I was wrong.

For LLMs, that’s not irrelevant, but it’s secondary.

A faster GPU and more modern cores do help, but only **after** the model fits comfortably in VRAM. VRAM isn't half the battle, it's more like 90% of the battle. If it doesn’t fit cleanly, more compute doesn’t save you.

This flips the usual upgrade logic on its head:

> For gaming: more GPU horsepower
> For LLMs: more VRAM

In this respect there is some hope for the average home enthusiast - it's not always the latest and greatest (and most expensive) setup that wins. An older card with more VRAM is potentially more desireable than a current-gen card with less. (There's a reason why the market for used RTX 3090's is insane right now - it's a six-year-old card, but it's all about that sweet, sweet 24GB of VRAM.)

## Noise, Heat, and Practical Reality

Another thing that doesn’t show up in benchmarks is what it's like to  **live with the hardware**. Running LLM workloads spins up fans, increases case temps, and adds background noise. My GPU fans spin up with every prompt and then back down again after the response. It’s not unbearable, but it’s noticeable. And if I'm hitting it fairly hard, my office gets perceptibly warmer. Gamers are no stranger to this sort of thing, but this is one of those small quality-of-life factors that can add up over time. Make sure these things won't bother you or, if they do, can be mitigated.

## What Actually Matters (So Far)

After a lot of trial and error, here’s my current mental model:

1. **VRAM is the primary constraint. System RAM is a fallback for certain use cases, not a general solution.**
2. **“Can it run?” is a much lower bar than “Is it usable?”**
3. **TTFT is just as important as tokens/sec**

## Revisited: Privacy, Cost, Sustainability

### Privacy

Clear win. Everything stays local, no ambiguity.

### Cost

Mixed so far: no subscription fees is a plus, but hardware constraints are very real and upgrading for more VRAM gets expensive quickly. Energy isn't free, either, as I discuss next.

### Sustainability

A home PC GPU like mine doesn’t just consume power under load—it consumes a non-trivial amount even when idle (~50–100W). If I leave this system on all day “just in case I want to use AI,” that’s a meaningful amount of energy consumption—even if I only run a handful of prompts. That complicates the sustainability argument, because that's potentially a fair amount of waste (not to mention some cost - a constant 50W idle load equates to $55 a year for me). This is where cloud-scale AI has advantages - those machines are kept busy constantly and can be scaled up or down as needed.

I can mitigate this somewhat by turning systems off when not in use and only running models on-demand. But that introduces start-up friction as I mentioned earlier.

My setup uses relatively clean energy, but idle power and lower utilization are real factors. We're not clearly winning on sustainability yet.

## Where This Leaves Me

At this point:

* Yes, I *can* run local LLMs on my existing hardware
* But I’m operating within very real constraints
* And those constraints are shaping every decision going forward

The next question is:

> **What actually runs well within those constraints?**

"Technically works" and "usable" are very different things. It's clear to me now that Local LLMs aren't so much about capability - because it largely *can* be done - but more about constraints and the experience that can be delivered within those constraints. When local LLM performances degrades, it isn't often a smooth, gradual degradation - everything works well until it suddenly works terribly. Running local LLMs seems to be an exercise in overcoming one constraint after another, and you really need to be aware of your specific limits and how an LLM is interacting with your hardware. A key skill to develop is how to ride right at the edge of those limits without exceeding them - something I didn't fully appreciate at the start of this journey.

## Next Up

In Part 3, I’ll walk through the software stack I ended up with (so far) — what's worked, what didn’t, and where things currently stand.
