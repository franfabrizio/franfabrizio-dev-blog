---
title: "Adventures in Local LLMs Part 4: Choosing a Model for Your Setup"
date: 2026-05-01T00:00:00-05:00
draft: false
toc: true
tags: [
  "llms",
  "local-ai",
  "homelab"
]
---

## Introduction

In Part 3, I walked through the software stack I ended up running — Ollama, Open WebUI, paperless-gpt, and a handful of other tools — and how much of it felt like a science experiment. The stack worked, mostly, but it was only half the equation. The other half is the model itself.

Once you've got your hardware constraints figured out (we covered those exhaustively in Part 2) and your software stack in place, you're left with a deceptively simple question: *which model do you actually run?*

The answer is not obvious. There are hundreds of models out now, dozens of quantization variants, and each one makes different trade-offs between quality, speed, context window, and tool use. Some models are great for conversation and terrible at coding. Others are built for structured outputs and terrible at open-ended chat. Some models look great on paper and fall apart once you actually try to use them with your hardware.

In this post, I'll walk through how I approached evaluating models for my use cases, what I tested, what surprised me, and how the model I chose for each job isn't necessarily the one I expected. I'll also share a few practical tips at the end for how you can evaluate models yourself, since the right choice depends entirely on your constraints and your needs.

And yes — I did buy new hardware between Parts 2 and 3, and it changed the model selection problem fundamentally. More on that soon.

## How to Evaluate Models

Before I started testing models, I needed a framework. Not some elaborate benchmarking suite — just enough structure to make the results meaningful.

I ended up with three categories of evaluation, one for each of my primary use cases:

### Chat Quality

This was the easiest to assess. I'd run my usual prompts through each model and look for things like:

- Does it actually answer the question, or does it dance around it?
- Is the tone reasonable? (I've seen models that are overly enthusiastic, overly cautious, or just bizarrely formal)
- Does it hallucinate facts? I'd ask it questions where I knew the answer.
- How does it handle follow-up questions? Did it remember context, or did it lose the thread?

The thing about chat quality is that it's subjective. One person's "great model" is another person's "too verbose." I tried to focus on objective markers where I could — accuracy, coherence, helpfulness — but I'll admit there's a subjective component.

### Document Processing Accuracy

For this, I used paperless-gpt as my test harness. The task is well-defined enough that differences between models become apparent quickly: extract the correspondent, document date, document type, title, and tags from a PDF.

I fed the same set of documents through each model and compared the results. Document processing is a relatively constrained, structured task, so if a model can't do it well, it's a pretty strong signal that it's not going to be great for much else either.

### Coding Assistance

This was the hardest to evaluate. Coding assistants need to understand code structure, follow instructions precisely, and generate working code — all while being interactive enough to iterate with you. Local models on consumer hardware were... not great at this yet. But I still tried to assess:

- Can the model understand and explain existing code? (Easier than generating new code)
- Can it follow multi-step instructions without getting confused?
- Did it produce code that actually compiles/runs?
- How well did it handle debugging requests?

I was not looking for models that could write full applications from scratch. Even cloud-based models struggle with that. I was looking for models that could be useful partners in the coding process — and that was the bar.

### Performance Metrics That Matter

For each model, I tracked:

- **Tokens per second** — how fast it responds once it's started
- **Time to first token (TTFT)** — how long you wait before seeing anything
- **VRAM usage** — did it fit cleanly in VRAM or spill into system RAM?
- **Context window behavior** — how did it handle longer conversations or documents?

Those first two in particular are critically important, at least to me. If the model can't generate tokens at least as fast as I can read, it begins to get very tedious to iterate, especially on coding, regardless of how well the model performs. The TTFT is less critical but to a limit - at some point you just lose focus while waiting for a response.

## My Hardware Situation (Updated)

In Part 2, I described my baseline setup: a Ryzen 5900X with 32GB RAM and an NVIDIA 3060 Ti with 8GB of VRAM. That card was the limiting factor for everything. Models that didn't fit in 8GB spilled into system RAM, and the performance penalty was brutal — TTFT jumping from 3 seconds to 30+ seconds, tokens/sec dropping to unusable levels.

Long story short, after experimenting for a while, I decided to upgrade my GPU to an **RTX 5060 Ti with 16GB of VRAM**. This wasn't a random choice — at the current time, it's the cheapest consumer card that gave me modern CUDA architecture and 16GB of VRAM. The 3060 had 12GB but wasn't worth the upgrade cost. The 3090 has 24GB but costs a fortune, even used. The 5060 Ti hit the sweet spot for my budget. Now, I wasn't considering AMD cards but the software ecosystem around them is maturing to the point where I think I'd consider that now as well.

The difference with 16GB VRAM was night and day. Doubling my VRAM budget from ~5GB usable to ~13GB usable (I still had to leave room for the display and system overhead) meant I could run significantly larger models without spilling into system RAM. Models that were previously unusable because of VRAM spillover now ran at full speed. The TTFT for models that fit cleanly in VRAM dropped to seconds instead of tens of seconds. And most importantly, the models that can fit into 16GB, especially with newer quantization strategies, start feeling meaningfully close to cloud model performance for a lot more tasks than those that fit into 8GB VRAM. That's not to say that you can't get anything useful done with 8GB VRAM, you absolutely can -- you just need to adjust expectations and break down work into smaller pieces.

With 16GB VRAM, I could explore the 14B–20B parameter range without the performance penalty, which opened up a whole class of models that had noticeably better reasoning ability. But the real game-changer came when I started exploring **Mixture of Experts (MoE)** models — architectures where the model has tens of billions of total parameters but only activates a small subset during any given forward pass. This meant that a 35B-parameter model could run on my 16GB card just as efficiently as a 7B dense model, because only about 3.6B parameters were active per token. That changed the entire landscape.

## What I Tested

I tested a broad range of models across different parameter sizes and architectures. Here's the rough spread:

**Small models (under 7B parameters):**
- Qwen 3/3.5/3.6 variants (0.6B, 1.7B, 4B, 8B)
- Gemma 4 variants (2B, 9B)
- Llama 3 variants (8B)
- Mistral variants (7B)
- Phi-3 mini

**Medium models (13B–20B parameters):**
- Qwen 3/3.5/3.6 variants (14B)
- Gemma 4 14B (dense)
- Llama 3 variants (70B — quantized)
- Mistral Large variants

**Mixture of Experts models:**
- Qwen 3.6-35B-A3B (35B total params, 3.6B active)
- Gemma 4 12B-A3B
- Gemma 4 32B-A3B

**Large models (30B+ parameters):**
- Qwen 3.6 72B (heavily quantized)
- Qwen 3/3.5 72B (heavily quantized)

For each model, I tried different quantization levels — Q4 (4-bit), Q5, Q6, Q8 — and sometimes the full precision version if VRAM allowed. For the Qwen 3.6 models specifically, I also tried the newer **unsloth quantizations** (UD-Q4, UD-IQ3_S), which I'll get to shortly. These newer quantizations offered a quality-per-VRAM ratio that changed how I think about model selection.

## The Quantization Question

Quantization is one of the most important decisions in model selection, and it's the reason we're able to run larger models on consumer hardwarte at all. Quantization is the process of reducing a model's precision from 16-bit floats to 4-bit integers (or 5-bit, or 6-bit, etc...). It shrinks the model's memory footprint significantly, but at some cost to quality. The key is finding the right balance - small enough to fit and run efficiently while preserving quality enough to still be reliable.

Here's what I found. First, for the "traditional" quantization levels:

**Q4 (4-bit):** Models shrink significantly, fitting into less VRAM. This is the default quantization level that Ollama uses. But the quality loss is noticeable at times.

**Q5 (5-bit):** Good quality, but started to pressure VRAM usage. The difference between Q4 and Q5 was noticeable; the difference between Q5 and Q6 was marginal.

**Q6 (6-bit):** Slightly better quality than Q5, but noticeably more VRAM usage.

**Q8 and above:** Close to full precision, but significantly larger. Q8 was great when VRAM allowed, but for many models, the VRAM overhead was too much. The 72B model at Q8 wouldn't fit in 16GB VRAM at all, even with a small context window.

Along the way, I discovered alternative, newer forms of quantization. It took my a while to find these because Ollama hides quantization from the user in the name of convenience. This is great as a beginner because all the different quantization levels can get confusing, but at some point if you're going to maximize performance of your local LLM setup you'll need to venture into this area. In particular, there are the Unsloth quantizations. The one I found to be incredibly useful was:

**Unsloth UD-IQ3_S (ultra-dense informatic quantization):** Unsloth developed a new quantization scheme that applies information-theoretic quantization to reduce VRAM usage further while preserving model quality. The UD-IQ3_S variant on the Qwen 3.6-35B-A3B uses roughly the same VRAM as a standard Q4 quantization but retains significantly more of the model's capability. This was the key enabler that made the 35B MoE model viable on my hardware — standard Q4 would have been too lossy, but UD-IQ3_S preserved enough quality that the model actually works well in practice.

### From Ollama to LM Studio

But how do I unlock these alternative quantization methods for testing?

When I first started, I used Ollama the way everyone does — download a model, run it, and let Ollama decide which quantization to use (which is generally Q4/Q4_K_M). Ollama's model library is curated and convenient, but it only offers a limited set of quantizations. I was essentially evaluating models on their default quantizations without even knowing there were alternatives.

When I discovered unsloth's quantization schemes — UD-Q4, UD-IQ3_S — I realized I'd been holding myself to an artificial ceiling. I then switched my testing setup to LM Studio. LM Studio exposes a lot more of what's going on under the hood (it's great for debug logging too), and it allows me to download and test whatever quantizations I want.

This is where I left the nicely curated world of the Ollama model library and entered the wild west of Hugging Face. If you stick with this local LLM hobby, you won't be able to avoid Hugging Face. It's the de facto hub for open source AI everything - models, datasets, apps, and so on. And I need to tell you right up front: the Hugging Face user experience is awful. It's a website that accreted its functionality over years without any overarching design plan. It caters to ML researchers and clearly not enough UX expertise has been present. It's incredibly hard to navigate and find what you're looking for, and the information is basically presented in a nearly-incoherent mess of spaghetti. Having said that, it's also indispensible - from Hugging Face I can download any GGUF quantization without the Ollama gatekeeping and compare a model at Q4 vs Q5 vs Q6 vs UD-Q4 vs UD-IQ3_S side by side, which will reveal meaningful differences.

For CLI fans, there's a huggingface CLI that helps avoid the need to wade through the website somewhat, which makes using HF slightly less frustrating for me.

## Results by Use Case

### Chat: Surprisingly, Smaller Models Were Fine

I expected that for chat, I'd need a model with enough parameters to handle the nuances of conversation. I was wrong.

The **Qwen 3 8B** was adequate for chat. It handled follow-up questions well, maintained context across a reasonable conversation window, and didn't hallucinate as often as I expected. The **Qwen 3.5 9B** was slightly better at nuanced conversation, but the difference was marginal.

The surprise was that even smaller models — the 3B and 5B variants — were usable for basic chat. They weren't as good at handling complex follow-ups, but for quick questions and casual conversation, they felt fine. This matters because it means I don't *always* need the biggest model and can have a specific model set up for quick chats.

The **Llama 3 8B** was solid too, but I found the Qwen models generally more responsive and less verbose in my testing.

### Document Processing: A Clear Winner

For document processing, **Qwen 3.5 9B** was the best balance of quality and performance. For most documents it would be indistinguishable from the Qwen3 14B model in terms of accuracy. For trickier documents with unusual formatting or mixed content, the 14B pulled ahead somewhat, but at a significant performance cost.

The smaller models (3B, 5B) struggled noticeably with metadata extraction. They got the basic stuff right but made more mistakes on document classification and date extraction.

This was useful because it gave me a practical rule of thumb: for document processing, aim for at least 7B–9B parameters, and 14B if you want more reliable results on diverse document types.

### Coding Assistance: Finally Workable

This was a disappointment for most of my testing. Most models that would fit into my VRAM could understand code and explain it, but generating new code reliably was another matter. In fact, I was about to give up on this use case until I discovered alternate quantizations.

The **Qwen 3.6-35B-A3B** at Unsloth's UD-IQ3_S was something of a revelation and is the model I now use for local agentic coding. It can explain code, identify bugs, scaffold new files, and set up project structure. It can also write code from scratch — it sometimes gets stuck in reasoning loops, often requires a few iterations to work out all the bugs, and still struggles with multi-file changes. But the quality of the code it produces is good enough that I use it for real work, not just experiments. This was a major milestone for my setup.

The MoE architecture + Unsloth quantization is key here. Even though the model has 35B total parameters, only about 3.6B are active per token. That means it runs on my 16GB VRAM with the same efficiency as a much smaller dense model, while having the total parameter count and quality preservation to actually be decent at coding.

One surprise was how much I disliked **Gemma 4 32B-A3B** for coding. It often got stuck in reasoning loops and overall was just a much more frustrating experience.

The **Qwen 3.5 9B** was surprisingly decent at code analysis, even though it wasn't specifically trained for coding. If you're just working on cleanup of an existing codebase, this might be a totally fine choice that will run more responsively.

I did try the larger models like 72B quantized — and while they were better at reasoning about code, they were often too large to run efficiently. The 72B model at Q4 quantization fit in my 16GB VRAM barely, with almost no room for KV cache. By the time I'd sent a few messages in a coding session, the context window had filled the available space and forced spillover into system RAM, destroying performance.

### Tool Use: Still Tricky

I tested tool use (models calling external tools like a web search API) across several models, as this is essential for agentic coding. The results were inconsistent. Some models supported tool use but broke with certain context window sizes. Others kind of worked but produced malformed output. A few didn't support it at all.

The Qwen 3.6 series was the first I tested with what I'd call reliable tool calling, and was a very noticeable improvement from the Qwen 3.5 series. It's still not 100% but it's reliable enough to not feel like friction during agentic sessions.

## Context Windows and KV Cache Pressure

For many use cases, a modest context window of 8K or 16K is more than enough. One thing that caught me by surprise was how aggressively the KV cache eats VRAM, especially with larger context windows.

Going from a 4K context to a 16K context is not a linear increase in VRAM usage. It's exponential, because the KV cache stores the model's internal representation of the conversation across every layer. For a 14B model with 48 layers, the difference between a 4K and 16K context was enormous. 

This means that **a model that fits cleanly in VRAM at the start of a conversation can spill into system RAM partway through**, and you won't know it until performance suddenly degrades.

For MoE models specifically, the KV cache is smaller relative to total parameter count since only the active experts' parameters need KV cache entries. This is one reason why the Qwen 3.6-35B-A3B runs so efficiently for coding — its KV cache footprint is more like a 7B dense model, not a 35B model. That's a significant advantage for VRAM-constrained setups, and it's critical for agentic coding, where I find I need a minimum of 128K context window to be effective. This would simply be impossible on my hardware without the MoE optimization.

My practical approach:

1. Estimate VRAM needs: model weights + KV cache for your expected context + ~0.5GB overhead
2. Leave a buffer. Don't plan for 100% VRAM utilization. Leave 1–2GB headroom.
3. Monitor VRAM usage during longer sessions. It's natural for tokens/sec to slowly degrade over a long session, but if you notice tokens/sec dropping suddenly mid-conversation, your KV cache is spilling.

## Practical Model Selection Guide

With the pace of change in this space, this entire blog post will likely be obsolete by the time you read it. So, I want the takeaway to be not so much the specific models I landed on, but the process I used to get there, so that you can replicate it with whatever is the latest and greatest of the moment.

If you're reading this and thinking about choosing models for your own setup, here's a framework you can use:

### Step 1: Know Your VRAM Budget

Subtract the VRAM your display and system apps use. That's your starting point. Then decide how much you want to leave as overhead. The remainder is your model + KV cache budget. Leave more than you think aside for KV cache, you'll need it!

### Step 2: Pick Models for Your Use Cases

Different tasks need different models. Chat needs conversation quality. Document processing needs structured output capability. Coding needs reasoning ability. Don't expect one model to excel at everything.

### Step 3: Test with a Constrained Task

Use something like paperless-gpt for document processing — it's a well-defined task with clear right/wrong answers. If a model struggles there, it's unlikely to be great for anything. Use it as a sniff test for a new model.

### Step 4: Check Tokens/sec and TTFT

Run a few prompts and measure the speed. If it feels slow, check if you're spilling into system RAM. If you are, try a smaller model or a higher quantization. My personal tolerance level is something like 25 tokens/sec for interactive use cases - below that and I get frustrated quickly. You may have more patience than I do!

### Step 5: Test Different KV Cache Sizes

KV Cache size is a big factor in quality and performance because once you fill KV cache, a model needs to start "forgetting" things to manage the cache. That oftens ends up being a bigger factor than just raw model performance, and sometimes it makes sense to choose a smaller model with a larger cache.

### Step 6: Don't Ignore New Quantizations

Standard quantizations (Q4, Q5, Q6) are well understood. But newer approaches like unsloth's UD-IQ3_S can offer better quality-per-VRAM than traditional quantization. If you're pushing against a VRAM wall, look into what unsloth or similar projects offer — the quality difference compared to Q4 can be substantial.

### Step 7: Consider MoE Models

Mixture of Experts models are what allows you to push the limits of local hardware. They offer large total parameter counts with small active sets, which means you can get the quality of a large model at the compute cost of a smaller one. The Qwen 3.6-35B-A3B is a particularly good example of this pattern done right — it's the model that finally made local coding assistance viable for me.

### Step 8: Iterate

Model selection is not a one-time decision. New models come out constantly. What works today might be superseded next month. Keep testing, and don't be afraid to switch models for different tasks.

## Conclusion

After all this testing, what have I learned?

1. **VRAM is still king.** Doubling from 8GB to 16GB changed everything. The model that fits cleanly in VRAM will always beat a larger model that spills into RAM. Even the 5060 ti, which has relatively modest memory bandwidth relative to higher-end cards, made a huge difference for me.
2. **One model doesn't fit all use cases.** Chat, document processing, and coding each benefit from different model characteristics. It's fine to run different models for different tasks — just be aware of the loading/unloading cost.
3. **Smaller models are better than I expected.** For many tasks, a well-quantized 7B–14B model is plenty. You don't always need the biggest model.
4. **Quantization matters — and is evolving fast.** Q5 is the sweet spot for traditional quantizations, but unsloth's UD-IQ3_S and similar newer schemes are changing the math. A model that wouldn't have worked at Q4 might work great at UD-IQ3_S.
5. **Context windows are a hidden cost.** The KV cache eats VRAM faster than you'd expect. Plan for it. MoE models help here since their KV cache is smaller relative to total parameter count.
6. **Coding on local LLMs is now workable.** The Qwen 3.6-35B-A3B was the model that finally crossed the line for me. It's not perfect, but it's good enough for real work. The MoE architecture — large total params but small active set — is a genuinely useful pattern.
7. **The landscape is moving fast.** Models I tested today may be superseded next month. Keep testing.

Local LLMs are viable for many use cases, but the experience is very dependent on your hardware, your choice of model, and your expectations. There's no one-size-fits-all answer, which is why the testing matters.

## Next Up

This is the last installment in the Adventures in Local LLMs series. I've covered the hardware constraints, the software stack, and the model selection. This is enough to get you to a competent, useful local LLM setup. If you stop here, you'll still be able to get the privacy, cost and sustainability benefits that local LLMs can provide.

Having said that, we've really just scratched the surface of the local LLM world. Since I finished this series, I've already explored many other interesting avenues. One was to create a dedicated AI server independent of my main PC so that I could have always-on AI inference just waiting for when needed. I used old datacenter hardware to build a dual GPU, 48GB VRAM server for about $1000! ([More on that build here](/posts/am4-tesla-p40-buildout/).) I still use the main PC setup we created in this series for agentic coding because the newer GPU hardware is more performant for tight iterating, but for general chat and batch processes, my new AI server is working great.

Another interesting area I've been exploring lately is the tooling layer that can sit between AI clients and local LLMs. This includes Retrieval Augmented Generation (RAG) systems for injecting local knowledge, plugins to manage context compression in flight, persistent memory layers that can remember and re-surface important details as needed so each new session doesn't feel like starting over, and MCP-based integrations with other local systems to further integrate AI into your homelab. I've been using MCP integrations to have AI help me document my homelab in my Bookstack instance, for example. I'll blog about all of these as I find time, but the TL;DR is that I am learning that getting to the right local model is just the beginning - to achieve true usefulness requires curating an entire AI ecosystem tailored to your homelab.

Thanks for following along, and happy local LLM adventures! I'd love to hear about what you're learning in the comments!

---

*This post is part of a series on local LLMs:*

- [Part 1: I Really Didn't Want to Do This](/posts/adventures-in-local-llms-part-1/)
- [Part 2: The Hard Truth about Hardware](/posts/adventures-in-local-llms-part-2/)
- [Part 3: The Software Stack is a Science Experiment](/posts/adventures-in-local-llms-part-3/)
- [Part 4: Choosing a Model for Your Setup](/posts/adventures-in-local-llms-part-4/) *(current)*
