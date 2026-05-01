---
title: "How Generative AI and LLMs Actually Work: Everything I Wish I Knew When I Started"
date: 2026-04-30
draft: false
toc: true
---

When using modern generative AI, it can sometimes feel a little magical. How does it know so much--and sound so convincingly human? Ever wonder how it works? Ever wonder how the AI can not only give you a response that makes sense but also seems to "understand" what you meant, knows details you didn't actually type in, seems to be an expert on every topic under the sun, and sometimes seems to know what you really were wanting it to do better than you did yourself?

This post is here to help answer these questions. I'm not a GenAI expert, and this post isn't going to be an exhaustive technical deep dive. However, one of my philosophies throughout my career has been to try to understand at least at a conceptual level what's happening under the hood of the tools I use every day. Take 20 minutes to read this post and you'll get a primer and a handy reference framework of how modern generative AI works. I'll also draw on an analogy from everyday life - cooking - to help cement these concepts.

I may gloss over some details for the sake of quick understanding (or because there's a lot of stuff I haven't learned myself yet!), but I'll leave you with a foundational mental model to help generative AI feel less like magic and more like the sophisticated computational technique that it is.

## What Do We Mean by Generative AI?

AI is a topic within computer science that dates back to the 1950s, almost as long as computers themselves have been around. Making computers act more like humans was one of the first sub-disciplines of computer science to emerge. And one of the first forms of AI was Machine Learning (ML). ML aimed to give computers the capability to "learn" without being explicitly programmed but rather through experience, like a checkers program that gets better over time by observing the moves it and its opponent made and remembering how effective or ineffective they were, and then using that information to inform future moves.

In traditional Machine Learning, humans usually configure the system with "features" - the important characteristics of the experience that should inform the learning. For instance, when learning checkers, it's important to observe whether a move helped you become a king, capture an opponent's checker, or led to the loss of your own checker. A different form of ML, Deep Learning, aimed to give computers the ability to figure out what observations were important autonomously without requiring a human to configure them.

Deep Learning does this through the use of multi-layered neural networks to learn patterns in data. Deep Learning can discover patterns that are very hard for humans to describe or even detect. Early layers of a neural network can detect simple patterns and relationships between data, but as you go deeper through the layers the relationships and patterns that are uncovered become more and more distant and abstract. In checkers terms, an early layer might see a pattern like "if I move my checker to the last row of the board on my opponent's side, it becomes a king" whereas a deeper layer might uncover a pattern like "if my first move is to move a checker from G1 to F2 and my opponent then moves B7 to C8, I need to move F2 to E1 or else that checker will be captured by the tenth round in 73% of games." Humans would be extremely unlikely to detect a pattern like that, but multi-layered neural networks excel at that sort of thing.

Now that we understand machine learning, let's talk about Generative AI. The simplest definition of Generative AI, or GenAI, is AI that generates new content - text, images, audio, video, code, and so on. Modern Generative AI generally uses deep learning to learn from past content in order to generate new content. This is how an AI like ChatGPT works.

Narrowing in on text-based Generative AI, let's go down another historical lane of AI, Natural Language Processing (NLP). NLP also arose in the early days of computer science and has remained an important area of research. NLP aims to help computers understand human language by enabling them to read and discern the meaning of text, translate between two languages correctly, and most pertinent to our topic today, generate new text. In the past, NLP techniques for generating text tended to be heuristic or statistics based, using probabilities to figure out what should come next. However, over time deep learning has become the primary approach for modern NLP. NLP is what powers capabilities like spam detection, sentiment analysis, language translation - and Large Language Models.

Bringing this all together, when we say GenAI, what we're saying is that we are using deep learning to learn from existing content in order to create new content. When that content is text, we're leveraging modern NLP to do so. Other types of content represent the merging of other computer science disciplines with deep learning, like turning a PDF into digital text via OCR (computer vision), creating audio (speech synthesis), or creating video (computer vision + temporal modeling). You may also hear the term "multi-modal", which refers to AI systems that combine these capabilities to generate multiple types of content.

For the rest of this post I'll focus on text generation using Large Language Models since this is the most prevalent form of Generative AI today, but much of this content is applicable to other types of GenAI as well. Because I am focusing on text in this post, I will sometimes use "model" as a shorthand for Large Language Models, but in the world of Generative AI the term "model" can refer to any sort of domain, such as an image generation model or a character recognition (OCR) model.

## An LLM Is Not Like Other Software

Large Language Models, or LLMs, are at the very heart of text-based generative AI. An LLM is essentially a very large mathematical function that has learned patterns of how language works by examining huge amounts of text, and can use that knowledge to generate new text based on a prompt from a user (it also uses its knowledge to understand the prompt itself!)

It is easy to make the mistake of thinking of an LLM like traditional software, but it is a category of software that has different characteristics than say an application or a database.

A traditional program:

* is written by a developer (or an AI acting in the role of a developer)
* contains explicit instructions
* executes logic and produces deterministic outputs given the same inputs

An LLM is different. A model encodes the patterns it is learning from examining text during a process called *training*, and it does so as a large set of numbers called *weights*. The weights are what represents all that learning about which words tend to follow others, how sentences are structured, and how ideas relate. A model is essentially a giant mathematical function made up of billions (or trillions) of learned parameters.

When you give a model input, it doesn’t “look up” an answer or execute logic the way traditional software would. Through a process called *inference* it passes the input through many layers of the learned weights, progressively generating output based on learned statistical patterns. Because the model produces probabilities rather than fixed answers, the final output can vary depending on how those probabilities are sampled. It doesn’t just pick the most likely next token--it uses those learned patterns to understand context, like recognizing “dog” is an animal and that a verb probably comes next in a sentence. It then generates a response that fits both the meaning and the structure of what you wrote.

One interesting quirk: models don’t work with words directly. They work with tokens, and all input is broken down into these tokens before the model processes it.

### Training vs. Inference

I introduced the terms training and inference in the previous section, and it's worth taking a moment to do a direct comparison, because you will hear these terms a lot.

Training is where the model learns patterns from massive amounts of text. It's hugely expensive - these are multi-datacenter-scale operations. Thankfully, it only needs to happen rarely, when a model gets "refreshed" with new or additional source text.

Inference on the other hand is when you use the trained model to generate outputs, which means inference happens every time you send a prompt. It's not a "cheap" operation, which is why we need nice GPUs to do it, but it's certainly many orders of magnitude cheaper than training.

## Tokens: How Models See Text

Models don’t see words the way humans do. They break everything into **tokens**, which are small chunks of text — sometimes whole words, sometimes parts of words, and sometimes punctuation.

Examples:

* “hello” → one token
* “unbelievable” → might be multiple tokens
* punctuation and spaces also count

When you interact with a model:

* your input is converted into tokens
* the model predicts the next token
* then the next
* and so on

This is what people mean when they say:

> “It’s just predicting the next token”

That’s technically true—but the important part is that the model has learned **very rich patterns** about language, so that token prediction is happening with full awareness of billions of learned relationships, which is why the output feels meaningful and not like autocomplete.

Another important aspect of tokens is that they've become the de facto measure of the *business* of LLMs. Tokens are also the currency of commercial LLMs, usage is metered and billed based on token count. Because more tokens == more cost, tokens/performance has become a measure of the efficiency of a model.

## Why It Sometimes Feels Like Actual Thinking - It's a Matter of Scale

You can ask a model to:

> “Compare democracy and communism”

…and get a structured, thoughtful answer.

What’s happening?

The model isn’t “thinking” in the human sense. Instead, it has learned patterns like:

* how comparisons are structured
* what kinds of arguments are typically made
* how to organize multi-part responses

So it produces something that *looks like thinking and reasoning* because it has seen many examples of reasoning during training. Models are trained with amounts of data that are really hard for the human brain to comprehend - not just billions, but in some cases hundreds of billions or in a few cases even *trillions* of weights. What appears to us as thinking is really just pattern matching at an unfathomably large scale.

### Model Size (Parameters)

You’ll often see models described like:

* 7B
* 13B
* 70B

This refers to the number of **parameters** (weights). More parameters generally means more capacity and higher quality, at the cost of more memory and computation required. Bigger models are usually better—but only if your system can run them efficiently. In the cloud with effectively limitless capacity, this is less of a concern. If you're running LLMs locally on the other hand, it becomes a balance between size and usability.

### Quantization: Making Models Fit

Most models are too large to run directly on consumer hardware, so the field has developed techniques for making models smaller. Quantization is the process of reducing the precision of the model’s weights to make it smaller and faster. Think of it like storing numbers with fewer decimal places.

Quantization reduces memory usage and improves speed, but it can reduce quality. Quantization can be applied to various degrees, so you'll see things like 4-bit vs 8-bit quantization (more bits preserves more precision).

There are also many different strategies for applying quantization to a model. The most common one you'll encounter with open source models today is GGUF K-quant (what Ollama uses for its models, often named with a _K suffix on the model name), but there are others like SmoothQuant, used by some models like llama3.

## Context Window: A Model's Short-Term Memory

The **context window** is how much text the model can consider at once, and it's an extremely important concept to understand when using LLMs, because it explains a lot about the output you're seeing.

The context window includes, your current prompt AND all of the previous messages in this conversation AND any additional data that has been provided to the model when the session started or during the conversation.

In some systems, especially reasoning-oriented ones, intermediate steps (sometimes called “reasoning tokens”) also consume space in the context window. Context windows can get filled astonishingly quickly.

A larger context window allows longer conversations and working with larger amounts of data, but also uses more memory and can slow the inference process down.

### What Happens When the Context Window is Filled?

Once you exceed the context window, the model starts to “forget” earlier information - as new tokens come in, old ones get pushed out of the context window. This is why you'll notice that models seem to be a little forgetful and need to be reminded of past details. When using LLMs, you cannot trust that they still "know" what they knew a few minutes ago. It can be infuriating at times!

Many tools have techniques which scrub or compress the context window to remove filler and less important information to try to stretch the window out for as long as possible, but you'll find yourself filling context windows constantly.

## Transformers, GPUs and the Web: the Keys to the Generative AI Boom

A model isn't just a pile of weights--it’s those weights organized according to a specific computational *architecture* that defines how input is processed and output is generated.

In computer science terms, an architecture is a specific pattern or design for organizing computational systems, whether that's software (monolith vs. microservices), hardware (x86 vs. ARM), or machine learning models.

You may have heard of the term "transformer" in the LLM space. Transformers are the dominant architecture for today's LLMs. Unlike earlier approaches, which process text strictly one token at a time, transformers can evaluate all tokens in a sequence together within each layer. This allows them to capture relationships across the entire input, even though they still generate output tokens sequentially. A helpful way to think about it is the difference between reading a paragraph one word at a time versus being able to consider the entire paragraph at once.

This parallelism has several major benefits over sequential approaches:

* Faster (especially on modern hardware like GPUs)
* Easier to make connections between tokens that are far apart — better for capturing higher-level concepts and relationships
* Able to work with much larger context

With sequential models, these desirable characteristics are difficult to achieve because you're considering one token at a time and must carry forward a compressed summary of everything you've seen so far. As the sequence grows, it becomes harder to retain important details from earlier in the text.

Transformers take a different approach. Given a chunk of text (within the model’s context window), transformers consider all tokens together and score each token-to-token relationship in that set. They then use a mechanism called attention to assign weights to these relationships, which allows the model to focus on the most relevant connections. This process is called self-attention and is repeated across many layers, with each layer refining how the tokens relate to each other and building increasingly rich representations of the text. Attention is how the model homes in on what's truly important about the input text.

### Probability and Sampling: Why Outputs Vary

At each step, the model doesn’t pick the next word—it produces a list of possible next tokens with probabilities.

The system then samples from that distribution.

Low randomness → more predictable, repetitive
High randomness → more creative, but less reliable

This is why the same prompt can produce different outputs—and why settings like *temperature* matter. Temperature controls how random the output tokens are. Low temperature = most common, repetitive. High temperature = more creative and surprising, sometimes out of left field. Related to temperature are some other tuning parameters like top_k, which is straightforward and means "choose from only the k most likely tokens", and top_p, which is more common now and means "pick enough tokens to reach probability p".

This sampling behavior is why outputs vary even when given the same input prompt.

## Hardware and Source Data: the Other Ingredients Fueling LLMs

In a very happy "coincidence", there's another computer science domain that depends upon massive parallelism and has seen rapid growth: computer graphics. Driven in part by the popularity of gaming, GPUs have steadily become ever more powerful parallel computation engines. In computer graphics, this allows the GPU to calculate and render all of those frames full of millions of pixels fast enough to produce smooth, detailed 3d graphics. But performing simple mathematical operations over matrices of millions of values in parallel is not just useful for graphics. This is exactly the kind of computation transformers rely on—large matrix operations performed in parallel. (It's also why GPUs became essential for crypto mining).

The web also played a crucial role by providing massive amounts of publicly available text data. Modern language models are trained on enormous datasets, much of which originates from websites, forums, books, and other digital content. Without this scale of data, the capabilities of these models would be far more limited.

The transformer architecture, modern GPU hardware, the availability of massive datasets, and refined training techniques have combined to produce a massive increase in the speed, quality, and capability of language processing, leading to the current AI boom. Today, nearly all modern LLMs are based on this transformer architecture. Alternatives are beginning to emerge, but transformers remain the dominant approach for now.

## A Useful Analogy: The Recipe

I find that a helpful way to think about all of this is the domain of cooking.

In cooking, there are some common techniques - sequences of steps - like:

- Mix -> Shape -> Bake
- Chop -> Coat -> Fry
- Combine -> Simmer -> Reduce

We also have recipes. A recipe is essentially a snapshot of knowledge - it's the author's way of capturing "I know that if you take certain inputs and apply a specific process, you will produce a desired result."

A key point is that the same technique can be used with different inputs and lead to different outputs.

* Given the steps Mix -> Shape -> Bake:
  * If you input Flour + Sugar + Butter you get Cookies
  * If you input Flour + Yeast + Water you get Bread
* Given the steps Chop -> Coat -> Fry:
  * If you input Potatoes you get French Fries
  * If you input Shrimp or Vegetables you get Tempura
* Given the steps Combine -> Simmer -> Reduce:
  * If you input Tomatoes + Herbs you get Marinara Sauce
  * If you input Meat + Vegetables + Broth you get Stew

And finally, a recipe isn't very useful without a chef to execute it. That chef must be proficient with the specified cooking technique in order to take those ingredients and produce a quality finished product. Let's consider a baker - a baker has lots of experience with mixing and shaping doughs and using ovens, so they can execute the baking pattern regardless of the specific ingredients and produce a quality result - they can make bread and pastries by applying their same skillset.

What does all of this have to do with LLMs? Let's connect the dots.

* Cooking Technique -> Architecture. A specific process for turning input tokens into output tokens.
* Recipe -> Model. A learned set of patterns that defines how inputs are transformed into outputs.
* Ingredients -> Prompt. The specific inputs provided for a task.
* Chef -> Engine. The system that executes the model using a given architecture.

Let's apply this analogy to a typical LLM interaction.

1. When you give your chatbot or agentic coding assistant some source code and a request to look for bugs, you're providing ingredients for a recipe.
2. Let's say your chatbot is connected to Ollama, which is built on top of llama.cpp. llama.cpp is an engine that implements the transformer architecture. If we treat the transformer architecture as our “baking” technique, then llama.cpp is the baker that knows how to execute it.
3. Let's also say that you've configured Ollama to use the qwen3.5 model. qwen3.5 represents learned knowledge from having examined massive amounts of source code, along with discussions about that code and the bugs it contained. It also reflects patterns learned from developer conversations about what it means to “look for bugs” and how to respond to that request. In other words, qwen3.5 is a (massive) recipe - it represents how specific words and concepts are related to each other and the outputs that result from such patterns.
4. llama.cpp, our baker, loads up the qwen3.5 recipe and, because it knows how to execute the baking process (in other words, how to apply the transformer architecture), it takes your prompt and processes those tokens through the model, producing a set of output tokens that provide useful advice on the bugs in your code and how to resolve them.

I know your code is always perfect, but let's pretend it had a couple of common bugs, say an off-by-one logic error and a missing semicolon. Those are very common bugs - qwen3.5 probably saw millions of examples of each during training - so it recognizes those patterns. It also has seen millions of examples of how those bugs were fixed, and it has associated the bug patterns with their corresponding fix patterns. Because of this, qwen3.5 produces a stream of output tokens which gives you very good advice on the bugs in your code and how to resolve them.

Why do different models produce such different answers to the same inputs? After all, we said modern models are essentially all utilizing the transformer architecture--they're all baking recipes. Shouldn't the output be very similar?

It ultimately comes down to the quality of the training data and the training process. qwen3.5 was trained on massive amounts of source code and developer conversations about bugs. Another model, say gemma4, was trained on _different_ massive amounts of code and discussions, so it learned a somewhat different pattern than qwen3.5 did, and that's why given the same source code and request to find bugs, the two models may not find the same bugs.

qwen3.5 is your grandma's meatball recipe and gemma4 is the meatball recipe from the mom and pop Italian joint on the corner. They might both make good meatballs, but they're not the same, because they didn't learn how to make them the same way. Or maybe the corner place's meatballs aren't that good - in that case, the model may not have been trained that well for that task.

But of course your grandma's meatballs are your favorite, right?

## Putting It Together

Let's sum up the key themes of this post:

* A model is a learned function (weights) with an architecture (e.g. transformer) that defines how those weights are used.
* Models don’t “know” things. There's no "database of facts". They generate patterns based on similar patterns they were trained on.
* Outputs are probabilistic, not deterministic.
* They run at a scale that is so large it is hard to comprehend, giving them so much pattern-matching ability that it can seem like they are thinking.
* Models have a limited memory - they will start to forget things they previously understood as the conversation and data size grows larger.
* If all else fails, fall back to the cooking analogy to see if that helps with comprehension!

Phew, we made it! Pat yourself on the back. You now have enough of a mental model to reason about model behavior instead of guessing, and understanding these foundational LLM concepts will make you much more effective at choosing the right model, setting realistic expectations, and understanding the output you're getting.
