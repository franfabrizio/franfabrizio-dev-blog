---
title: "Adventures in Local LLMs Part 3: The Software Stack is a Science Experiment"
date: 2026-04-29
toc: true
---

## Introduction

After working through the hardware constraints, I had a pretty clear next step:

> Install the “standard” local LLM stack and see how it feels.

If you spend about five minutes in this space, you’ll quickly converge on a default answer:

* Use Ollama to run models
* Pair it with Open WebUI for a UI

That’s the path I took.

And to be fair—this stack **does work**. But it also exposed a lot of the rough edges that weren’t obvious until I lived with it for a bit.

## Why I Started with Ollama

In my first-pass research, I concluded that Ollama was essentially the default tool in this space (turns out it's more nuanced than that, which I'll get to). This is for a number of reasons:

* Simple install
* Huge model library
* Clean CLI
* Works cross-platform

My GPU is on my Windows 11 machine, and while I run both WSL and Docker on that system, I was worried that additional layers of abstraction would have performance impact, so I wanted something that worked on Windows 11 natively. Ollama checked that box.

After a quick and straightforward install, ollama was running and I was ready to download and run my first model. It's as simple as:

```bash
ollama run llama3
```

I'm not gonna lie, the first time I did this and then asked a question and got an answer, it was a thrill to see it actually working so quickly!

Ollama is a great way to get started quickly. It abstracts away a lot of the messy and tedious parts for you. For quick experimentation, it’s genuinely great. For long-term production, it's not so great, at least not out of the box, and we'll explore why in a moment.

It took me a while to realize this. The recommendation to "just use ollama" is so ubiquitous, I accepted it on face value without stopping to really understand what Ollama *is*. So let's explore the local LLM software stack and examine Ollama's place in that stack a bit more deeply.

To make useful local LLMs a reality, you need:

1. **A large language model**. A model is a giant mathematical function trained on massive amounts of text to learn how words, ideas, and sentence structures relate to each other (that's the Language part of LLM). Given input, it doesn’t just pick the most likely next word—it uses those learned patterns to understand context, like recognizing “dog” is an animal and that a verb probably comes next in a sentence. It then generates a response that fits both the meaning and the structure of what you wrote. (This may sound like simple prediction, but we'll explore in a future post how the function's learning is rich enough that it can produce something that looks a lot like reasoning.)
2. **An engine to run that model**. You need an executable that implements the architecture the model was trained for and knows how to apply the model's weights to turn inputs into outputs.
3. **A layer to manage models and their execution**. Something needs to load a model, invoke the engine on your hardware, pass in inputs, and handle the outputs.
4. **A user interface**. We need some sort of interface to allow a user, or an agent acting on their behalf, to submit input prompts and retrieve output responses. This can be as simple as a chat application or web app, or a coding assistant integrated into your IDE, or a plug-in to Paperless-NGX that is sending incoming documents to an LLM for automated metadata extraction. There's a case to be made that this should be split in two: standard human interfaces (e.g. chatbots) vs. programmatic interfaces that can act semi-autonomously (e.g. Claude Code). I don't think that's necessary for our purposes here, but it's a valid perspective.

So where does Ollama sit? Ollama covers #2 and #3—and even a very lightweight version of #4. It bundles a model engine with a simple toolset that handles downloading, loading, and running models for you, including automatically detecting and using your GPU when available. It provides an API that allows external tools to use those models. It also includes a basic built-in chatbot for direct interaction (that lightweight #4), which is best thought of as a testing and debugging tool, as it lacks features you’d expect from dedicated chat applications, such as conversation history and richer interaction controls.

Ollama is an incredible convenience because it's abstracting away a lot of complexity--dealing with model downloads, CUDA, drivers, flags, memory configuration, and more. Having said that, it also introduces a number of trade-offs that are worth understanding.

### The First Issue: Model Lifecycle and TTFT

The first issue I hit was something I didn’t anticipate mattering as much as it did:

**Models don’t stay loaded.** Out of the box, Ollama will load a model when you send a request, keep it resident for a short period (default ~5 minutes), then unload it to free memory. That means every “first request” has to pay the cost of loading the model into memory before any tokens can be generated. That's fine for ad-hoc usage, not great for AI-as-a-local-service.

With this configuration, what you'll experience is that the response to your first prompt will take 20 seconds or so as ollama first loads the model onto the GPU before it can begin processing. If it requires CPU spillover, this will be even longer. Your next prompt, if following soon after the first one, will then be just a handful of seconds until your response arrives. However, if you go do something else for a while and then come back to it, you'll need to pay that startup cost again. This quickly gets old when doing interactive AI. Ollama *does* support changing keep-alive behavior, but the behavior is inconsistent. The setting is not always exposed in clients, and some tools ignore or override it.

### The Second Issue: Model Switching Is Awkward

Models are optimized for specific use cases. Some are good at coding, others at vision, others at generating images, and so on. You'll almost certainly want to use more than one model. Simple on paper:

> “I’ll just switch between models depending on the task.”

You'll want to do this more often than you might think. Some use cases, like coding assistants, use different models depending on the task: they'll use a lightweight one for simple or background work, a medium-weight one for the bulk of the interaction with you, and maybe a heavyweight one when it has to do complex reasoning over multiple files or large codebases. This is easy to do when you're just hitting three different cloud APIs. With local LLMs, this is painful. Now, you're paying an *unloading* AND *loading* cost, potentially multiple times in a single prompt! This is unworkable in practice. You have two choices:

1. Use one model for everything even if it's not the perfect fit. This means you're waiting longer than you need to for simple tasks but don't quite have the reasoning power you'd like.
2. Use one model on your GPU only and a lightweight one on your CPU only. This at least gives you two layers of capability, but you need to ensure that the two models stay segregated in their own memory spaces.

### The Third Issue: Tool Calling (or Lack Thereof)

This one was a big surprise for me.

Modern AI workflows—especially for coding or automation—depend heavily on:

* Tool calling
* Structured outputs
* Agents

With local models via Ollama:

* Some models support it
* Some don’t
* Some *kind of* do, but break in practice

It turns out that there's not a universally adopted, mature way that models call out to external tools. This varies wildly from model to model, from no tool use capability at all to a deprecated or less-standard schema that your AI tool doesn't use to modern and well-suppored approaches. So instead of actually using external tools, sometimes you just get a weird tool-calling XML blob echo'ed back at you.

"Can you give me a listing of the current directory?"

sometimes results in an answer like:

```bash
     {
       "name": "bash",
       "arguments": {
         "command": "ls -la"
       }
     }
```

Other times, the tools you're using support external calls, but the model is unaware of this fact. For instance, my OpenUI instance has web searching capability, but sometimes models will say things like "As an AI developed by Microsoft with restrictions on browsing capabilities due to privacy concerns, I can't directly perform live searches or access real-time data from external websites."

Not helpful.

I also ran into cases where tool calling worked at some context window sizes, but not others with the same model. It's finicky, and without tool-calling, some use cases fall apart completely.

### The Fourth Issue: Integration Weirdness

Once I started integrating Ollama into other tools (like document processing or RAG setups), things got messy pretty quickly.

Some examples:

* Services thinking the model was “offline” when it wasn’t
* Models ignoring services' passed-in parameters or services overriding the model behavior I wanted
* Repeatedly resetting the tool to use a different model, once reverting to using a cloud model without my realizing it
* Some default or generated prompts from tools causing models to get stuck in reasoning loops

It's a bit more of a wild west than I expected, and I've spent more time debugging weirdness between tools than within them.

### At This Point: Is Ollama Still the Default?

This is where my perspective has shifted a bit.

Ollama is still:

* The easiest starting point
* Great for experimentation

But I’m no longer convinced it’s the *best* foundation for a serious local AI setup.

There are other options worth looking at, like:

* vLLM
* llama.cpp
* KoboldCpp

Each has different tradeoffs:

* Performance
* Flexibility
* Hardware support
* Complexity

I haven’t migrated off Ollama yet—but understanding its limitations has been important.

## What About the User-Facing Tools?

Now that we've covered getting our LLMs running locally, let's talk about the tools we use to interact with those models. Here's a quick review of my initial experiences with user-facing interfaces to this local LLM setup.

I needed a number of different tools for myself and my family, including:

* A standard chat interface (what my family will expect)
* Integration with my digital home office powered by Paperless-NGX, to improve and automate document ingest and cataloging
* Some form of coding assistant, terminal-based and/or IDE-integrated
* [Eventually] Integration with my Home Assistant environment, to interact with my smarthome. I haven't tried this yet, so I won't be covering tools for this use case here - look for a future post!

There will probably be more in the future, but these were my initial use cases. Here are the tools I've deployed to meet them.

### Open WebUI: A Good Home Base

Open WebUI is a self-hosted webapp that provides a ChatGPT-like rich interface for interacting with LLMs, typically backed by a tool like Ollama. It provides chat history, model selection, customizable prompts, and connections to external tools.

I've been pretty happy with Open WebUI. The interface is nice and modern, it exposes a lot of configurable parameters and prompts for tuning model behavior, it's multi-user, and it lets me configure and present convenient abstractions to my family members like "Everyday Chat" instead of "qwen3.5:9b". It's not all sunshine and roses, I do sometimes have trouble figuring out where or how to configure something, but the community is pretty substantial and it hasn't been that hard to get things sorted out.

One of the best things Open WebUI provides is a framework for creating external tool access for your models. I'm just beginning to experiment with this capability. My first step was to set up a self-hosted SearXNG service, which is essentially a meta search engine with an API that enables programmatic searches of the web. It's great for privacy, and it's also perfect for giving web search capabilities to LLMs. Open WebUI <-> SearXNG integrations are very popular for this reason. I did get it working and show that my models can call to it for real-time web search results, though I still have some tuning and education to do before I feel like I really understand it and am using it to its full potential.

Overall, I think Open WebUI checks the "standard chat interface" box rather well. I can chat with a variety of models and the interface feels productive and clean. It certainly doesn't solve the underlying constraints - a slow model is going to feel slow here. But when I've loaded a model that is well matched to my system's capabilities, chatting in Open WebUI feels pretty natural and not that different from chatting to a cloud LLM. Most importantly, with careful model selection, I could see my family actually using this--which is the ultimate litmus test.

Given that a large portion of my AI usage on any given day is asking random questions and going down rabbit holes with chatbots, it's no small win that this use case can feel natural with local LLMs.

### paperless-ai-next and paperless-gpt

These are the two most popular AI extensions for Paperless-NGX, my digital home office platform, and they take two different approaches.

* paperless-gpt focuses solely on document ingest. The goal is to be able to use LLM vision capability to OCR documents, and then standard LLM processing to extract key metadata, including the correspondent, the type of document, the document's date, a title for the document, and an appropriate set of tags.
* paperless-ai-next also provides those ingest capabilities, but adds a chat interface to interact with a single document and a new, more experimental RAG integration that tries to enable library-wide interactions (e.g. "show me my electric bill trend over the past five years").

paperless-ai-next is a fork of paperless-ai, which is a now-dead project that never realized its full potential. I am very interested in the idea of RAG chat - it would completely change the paradigm for how I retrieve information from my library - so I was hopeful when I discovered the new paperless-ai-next fork attempting to carry the torch. Unfortunately, the developer hasn't had time yet to focus on RAG, so it is still in the early, rather broken state inherited from paperless-ai. Long story short, it's a long way off from being good enough to be useful. As an ingest tool it can do alright with a lot of tweaking, but the more specialized and focused paperless-gpt does a better job and is what you should use if you're not trying to get chat-based interaction with your documents. For now, I've tabled paperless-ai-next and moved on to paperless-gpt.

paperless-gpt is certainly more polished and focused on maximizing the usefulness at ingest time. I've achieved results that are promising enough to turn on automatic ingest processing, but I will still be doing human review of every incoming document to check its work - it definitely makes mistakes and choices that I would not make. Overall I think it'll be a time-saver, though.

One of the nice side benefits of these tools is that they are a really effective way to test relative model effectiveness. It's hard to judge how good a chatbot's answer is relative to another chatbot, but it's much easier to evaluate one model's analysis of a document vs. another model. For that reason, I've been using these tools as a first-pass sniff test of various models. Document processing is a relatively prescribed and constrained task, so if a model can't do that well, it is unlikely to be very good at any of the other jobs I'd be asking it to do.

### Agentic Coding Assistants

Terminal-based agentic coding assistants like Claude Code, Codex, and GitHub Copilot are one of the most hyped applications of LLMs. Instead of writing code directly, you describe what you want, and the AI writes and modifies code for you. As a software development manager, I hear about them _constantly_.

It's been the "killer app" for LLMs so far - this is where corporations are willing to spend money to get high-end subscriptions to leading coding models for their developers, which is driving a lot of revenue to the big AI providers. As a professional leading a higher education/nonprofit-based development team, and as a personal hobbyist who likes to build new software, this business model is quickly going to get too expensive for both myself and my employer. Therefore, I am keenly interested in determining what's required to do effective agentic software development backed by local LLMs.

Claude Code and the other commercial agentic coding assistants can generally be coaxed into using a local LLM instead of their own proprietary cloud-based models, but that feels like a shaky foundation to build upon. It's obviously not in their corporate best interests to do that, so even if they don't disable it outright, they're certainly not going to put a lot of energy into making that mode work well. I did spend a bit of time testing a couple of them in my environment, but as I move forward, I will base my tests off of open source agentic coding assistants. Aider is an older one that is still useful, especially if you want your assistant to be more conservative and less autonomous. OpenCode, an open-source alternative to Claude Code, addresses my concern of vendor lock-in or local LLM shut-out. In fact, OpenCode has been helping me manage this blog over the past couple of weeks.

How well do these tools work with local LLMs? The jury is still out. Coding assistance is the hardest use case I've thrown at my local LLMs so far. I have to be honest and say that I'm not that optimistic yet. They have been decent at analyzing my existing code and understanding it, as well as being an administrative assistant for tasks like managing my repos and creating the scaffolding for new blog posts. But in terms of actually authoring code, what little I've tried seems to stress the models a lot, and so far I haven't found a model that runs well enough in my environment to feel responsive during iteration and is also a good coder. To be fair, I haven't tried all that hard yet. I have a couple of project ideas in mind that would be complete greenfield builds that would really test this use case, and I am looking forward to giving that a try sometime soon.

I don't want to give up on this use case as this would definitely qualify as a "killer app" for me, but at the moment this is the hardest use case to make viable locally.

### Agentic Personal Agents

I want to take a quick detour to address the latest trend in agentic AI. Over the past few months, the concept of the autonomous personal agent has exploded. It really gained momentum when OpenClaw went viral a few months back. The idea is an agent with a large library of skills and tools that can act not as a specialized domain-specific agent like a coding agent responding to your specific instructions at a very granular level, but as a more general-purpose autonomous assistant that can manage your email, your calendar, take over tasks on your behalf without detailed, step-by-step prompting ("Keep my INBOX free of spam")... one can immediately see the appeal of such a thing! However, it doesn't take long before you also realize with horror the potential security implications of an autonomous agent with broad access to your local system and your data with the ability to autonomously call out to external tools and/or impersonate you - arguably the most significant new attack vector in personal computing of the past few decades. For that reason, I am in no hurry to integrate these tools into my environment, and if and when I ever do, it will be with extreme caution.

### What's the State of Affairs?

It's a mixed bag across my use cases. I feel great about my current chat solution. I suspect with more fine tuning, my document processing workflow will also prove quite viable and useful. I'm less confident about agentic coding assistants - the complex reasoning required to write high-quality software systems might just be a bridge too far for local LLMs on typical consumer hardware.

On the other hand, models are getting ever more impressive on a per-parameter basis, so this situation could be markedly different by this time next year. As with everything else in this space, the pace of change is rapid, but today the limits of local LLMs are still very real.

## One Final Software Challenge: Observability is Surprisingly Bad

Before I go, there's one final nit I'd like to pick. I've demonstrated that there's a lot of tinkering and debugging required to integrate these tools. That's ok, I expect that as a homelab enthusiast. However, what I did not expect was how poor the observability was within this software stack. Rather than one weak link, it seems there has been very little focus on observability up and down this software stack. Want to see what exact prompts a tool is sending to Ollama? Not easy - from either end. Want to see what payloads the Ollama API is receiving? Not easy. Want to know what settings are available and how they're set? Not easy! Want to see or edit the default template that the tools are using to make prompts? Not easy! Everywhere I turned, things seemed harder to determine than they ought to be. ChatGPT suggested I do packet inspection on the wire. For real?

As just one example, many current models have thinking/reasoning capability - they iterate on a "draft response" with something akin to an internal monologue, refining and refining before they make their final response to the user. This can lead to more accurate answers, but it is resource-intensive and sometimes you want to turn it off in the name of quick replies. Most models provide some way to do so, but they're all different. And Ollama may or may not be able to set that parameter. And Ollama may or may not expose that to tools to let _them_ set that parameter. And tools may or may not expose that to the user for configuration. And tools may not document that they can do it, but they silently can. This was a hard enough problem to debug already, but with such poor observability, things get really desperate. You're left shooting off ten variations of think=false, no_think, reasoning=no, think=none, etc... into the ether in templates, in system prompts, in user prompts, at model startup, etc... with no ability to see what the tool or Ollama or the model is doing with that information. So, you just have to try one and then just ask the model another question and observe "is it still thinking or not?" Wash, rinse, repeat.

This is a painful and inefficient way to debug systems, and I'm surprised for such a cutting edge field that grew up in the era of observability, the situation was such a mess. I hope it improves soon!

## Conclusion

I hope sharing my experience with the software side of local LLMs gives the reader some perspective on the reality of running a local LLM service. As you can see, the whole space is still in flux. There are promising options everywhere, and there are rough edges everywhere. And at the end of the day, everything is ultimately constrained by what's happening at the bottom of the stack--can the models that your system can run efficiently measure up to the jobs you're going to throw at them? That answer is different for everyone, which is why my experience is not a replacement for your experiment. You've got to do the work and see what's possible in your world, but I hope that sharing my experience gives you a head start!

## Up Next

In Part 4, we'll get more detailed, exploring specific models and how well they're performing in my environment, including tweaks I've made to squeeze the most out of them that I can. I also need to fess up - I did buy some beefier hardware to try to expand the horizons of what is possible in my setup. All that and more, coming up!

---

*This post is part of a series on local LLMs:*

- [Part 1: I Really Didn't Want to Do This](/posts/adventures-in-local-llms-part-1/)
- [Part 2: The Hard Truth about Hardware](/posts/adventures-in-local-llms-part-2/)
- [Part 3: The Software Stack is a Science Experiment](/posts/adventures-in-local-llms-part-3/) *(current)*
- [Part 4: Choosing a Model for Your Setup](/posts/adventures-in-local-llms-part-4/)
