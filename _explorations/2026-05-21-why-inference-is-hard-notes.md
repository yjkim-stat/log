---
layout: post
title: "Notes on \"Why Inference is Hard\" — Artifacts, Quantization, and Engines"
date: 2026-05-21 10:00:00 +0900
description: >
  A reading of the YouTube walkthrough "Why Inference is hard.." into
  a structured map of the three layers that determine LLM serving
  cost — how the model is stored (artifacts, mmap), how it is
  compressed (standard / GGUF / AWQ / EXL2 / FP8 / NVFP4), and how
  it is run (llama.cpp / vLLM / SGLang / TensorRT-LLM / TGI).
tags: [inference, llm-serving, quantization, gguf, vllm]
toc:
  sidebar: left
---

The YouTube video
[*"Why Inference is hard.."*](https://youtu.be/B18zBnjZKmc) is the
best 15-minute version I've seen of the *mental map* a practitioner
needs before they read any single inference paper. It doesn't try
to be a definitive comparison — it tries to be the *taxonomy* you
can hang papers on. This post turns the video's chapter structure
into a structured note, with each chapter expanded just enough to
be self-contained and to make the next click obvious.

For a longer, periodically updated survey of where the inference
field is going, see the companion
[Inference Architecture report]({% link _reports/inference-architecture.md %}).
This exploration is the **entry point** to that report.

---

## 0. The Three Layers

The video implicitly partitions the inference problem into three
nearly independent layers. Most confusion in practitioner
discussions comes from collapsing them.

| Layer | What it controls | Vocabulary |
|---|---|---|
| **Artifacts** | How the model is *stored* and *loaded* into memory | safetensors, GGUF, mmap, weight tying |
| **Quantization** | The *numerical format* the weights (and sometimes activations) live in at inference time | INT8, INT4, GGUF (Q4_K_M…), AWQ, EXL2, FP8, NVFP4 |
| **Engines** | How tokens actually *flow through the model* at request time — prefill, decode, batching, scheduling | llama.cpp, vLLM, SGLang, TensorRT-LLM, TGI |

A model can be the same parameters end-to-end and behave radically
differently depending on which choice you make at each layer.

---

## 1. Artifacts and Loading

### 1.1 What an "artifact" is

A trained model on disk is a *collection of tensors* plus a small
amount of metadata (architecture name, tokenizer, special tokens,
RoPE settings, quantization metadata, etc.). The on-disk format is
not interchangeable across engines without conversion:

- **safetensors** — Hugging Face's modern default; tensor layout +
  JSON header, mmap-friendly, no Python pickle.
- **GGUF** — llama.cpp's single-file format; bundles weights,
  metadata, and quantization scales together. Designed for *local*
  CPU/GPU/MMAP loading.
- **engine-native** (TensorRT-LLM `.engine`, vLLM weight shards,
  etc.) — pre-compiled or pre-sharded for a specific runtime.

### 1.2 mmap — the single most underrated trick

`mmap()` lets a process treat a file as if it were memory: pages
are loaded on demand from disk by the kernel and reclaimed under
pressure. Three consequences for inference:

- **Cold start is cheap.** You don't pay for "load 70B weights into
  RAM"; you pay only for the pages the forward pass touches.
- **Multiple processes share the same physical pages.** Run 4
  inference servers from the same GGUF file and the OS deduplicates
  the resident set automatically.
- **It only works if the on-disk layout matches the in-memory
  layout.** This is why safetensors and GGUF are designed the way
  they are; pickled models can't mmap meaningfully.

The video's framing — *load is part of the inference problem* —
is the right one. On a single-user laptop, mmap is what makes
70B-class models feel viable at all.

---

## 2. Quantization

Compress the weights so they take less memory and less bandwidth at
the cost of (usually small, sometimes significant) accuracy. Five
families are worth distinguishing.

### 2.1 Standard quantization

Symmetric / asymmetric INT8 or INT4 with per-tensor or per-channel
scales. Cast the weights once, store the integers + a small set of
scale factors. Used as the baseline against which the more clever
schemes are compared.

| Bits | Typical recipe | What you lose |
|---|---|---|
| INT8 | per-tensor or per-channel scale | very little; often interchangeable with FP16 |
| INT4 | per-channel + zero-point | noticeable on reasoning tasks unless smarter scheme is used |

The "standard" baseline is what motivates everything below: at
INT4 the naive recipe hurts, and the rest of the field is people
inventing INT4 recipes that *don't* hurt.

### 2.2 GGUF (llama.cpp)

A *family* of quantization schemes packed into the GGUF file format.
Conventions look like `Q4_K_M`, `Q5_K_S`, `Q8_0` etc. — the letters
denote a block layout and a per-block scaling strategy. Key ideas:

- **Block-wise quantization** (e.g. 32-element blocks) with per-block
  scales — much better than per-tensor.
- **Mixed precision per layer** — important layers (attention output,
  embeddings) kept in higher precision; bulk MLP weights more
  aggressive.
- **CPU-friendly kernels.** GGUF was designed to run *fast on CPU*,
  which is why llama.cpp owns the local-inference niche.

### 2.3 AWQ — Activation-aware Weight Quantization

GGUF schemes are purely *weight-side*: they don't look at what
activations the model actually produces. AWQ does. The observation:

> A small fraction of weight channels are *salient* — they
> correspond to features the model uses a lot at runtime. Quantize
> those channels less aggressively and the accuracy hit drops
> sharply.

Saliency is measured by running a small calibration set through the
model and checking which channels have high activation magnitudes.
The result is INT4 weights that *behave* close to FP16 on real
inputs, not just on average. AWQ is the standard answer when you
want INT4 weights for a GPU inference server.

### 2.4 EXL2 (ExLlamaV2)

A *mixed-bitwidth, per-tensor* quantization scheme. Different
tensors are quantized at different bit-widths chosen automatically
to hit a target average bit-budget. Practical effects:

- Lets you ask for "this model at 4.5 bpw" and the system figures out
  which tensors should be 8-bit, 6-bit, 4-bit, 3-bit to make the
  average land there.
- Owns the GPU-only local-inference niche for power users running
  large models on consumer cards.

### 2.5 FP8 and NVFP4

The hardware-native end of the spectrum.

- **FP8** — 8-bit floating point, with E4M3 or E5M2 layouts. Native
  on H100 / H200. The smallest format that can run training-style
  numerics in-loop with no special tricks. Used heavily for both
  training and serving on modern data-center GPUs.
- **NVFP4** — Nvidia's 4-bit float (Blackwell-generation). Tightly
  packed micro-scaling format that makes INT4-class memory savings
  *with* the dynamic range benefits of a float, at hardware-supported
  speed. Where the high-end serving stack is moving.

The big-picture trend: **floats are catching up with ints at low
bit-widths**, and at the low end the question is becoming "which
4-bit format does your GPU have kernels for?" rather than "which
calibration recipe gives the best perplexity."

---

## 3. Inference Engines

The same quantized model behaves very differently across engines
because each engine makes different choices about scheduling,
batching, and KV-cache management. The video introduces the five
that matter; the differences sharpen once you think in terms of
**prefill** vs. **decode** vs. **serving**.

### 3.1 The two phases

- **Prefill** — process the input prompt. Compute-bound; one large
  matmul per layer over the whole prompt.
- **Decode** — generate tokens one at a time. Memory-bandwidth-bound;
  one tiny matmul per layer per token, dominated by loading weights
  and the KV cache from HBM.

A serving system is judged by how well it overlaps these two phases
across many concurrent requests.

### 3.2 The five engines

| Engine | Strongest fit | What it optimizes |
|---|---|---|
| **llama.cpp** | local / CPU / consumer GPU | mmap-friendly GGUF loading, CPU SIMD kernels, single-user latency |
| **vLLM** | data-center serving | **paged attention** for KV cache, continuous batching across requests, OpenAI-compatible server |
| **SGLang** | serving + structured generation | aggressive **RadixAttention** prefix sharing, frontend DSL for tool / JSON-mode workloads |
| **TensorRT-LLM** | Nvidia data center | ahead-of-time compiled engines, fused kernels for FP8 / NVFP4, max throughput at the cost of build-time flexibility |
| **TGI (Text Generation Inference)** | HF-stack production | batteries-included server, broad model coverage, streaming + token-level metrics |

The dimensions that actually move the comparison:

1. **KV-cache management.** vLLM's paged attention and SGLang's
   RadixAttention are the two interesting answers to "how do you
   share / reuse KV across many concurrent users?"
2. **Continuous batching.** Adding a new request mid-batch without
   restarting the GPU work — table-stakes for any serving system
   above llama.cpp's single-user scope.
3. **Quantization-format support.** TensorRT-LLM's edge is its
   hardware-aligned FP8 / NVFP4 kernel coverage; llama.cpp's edge
   is GGUF; AWQ is widely supported across vLLM / SGLang / TGI; EXL2
   is a power-user GPU format.
4. **Frontend ergonomics.** SGLang's structured-generation DSL and
   TGI's HF integration matter more for production than raw tokens/sec.

### 3.3 The mental model

The video's organizing point: **inference is not a single number**.
"Tokens per second" without specifying *which phase*, *at what
batch size*, *with which quantization*, *on which GPU* is mostly
folklore. The taxonomy above is the minimum set of axes you need to
make a comparison legible.

---

## 4. Putting It Together

A practical reading of the video's argument, restated:

1. **The artifact decision constrains the engine decision.** GGUF
   forces llama.cpp; engine-native compiled artifacts (TensorRT-LLM)
   foreclose ad-hoc model swapping; safetensors is the lingua franca
   that keeps options open.
2. **The quantization decision constrains hardware.** NVFP4 needs
   Blackwell; FP8 needs Hopper-class GPUs; AWQ runs almost
   everywhere; GGUF runs on CPUs that nothing else does.
3. **The engine decision dominates concurrency.** For a single user
   at low context, llama.cpp is usually plenty. The moment you need
   to serve many concurrent requests, the choice between vLLM /
   SGLang / TensorRT-LLM / TGI is the single biggest performance
   lever — *bigger than the choice of quantization*.

These three decisions are made independently but constrain each
other, and the most common mistake is to make them in the wrong
order.

---

## 5. What I'm Watching

- **Convergence on FP8 / NVFP4 as the default serving format.** The
  field has been INT4-curious for two years; hardware support for
  low-bit *floats* is reframing the question.
- **KV-cache reuse as the next big lever.** Paged attention solved
  *intra-request* memory; RadixAttention is the start of
  *inter-request* prefix reuse. The end-state probably looks like
  a content-addressable KV store shared across the cluster.
- **Compilation vs. flexibility tradeoff.** TensorRT-LLM-style AOT
  compilation gives the best throughput; vLLM / SGLang give the most
  ergonomic surface. Whether one side absorbs the other (eager
  engines adding AOT paths, or compiled engines exposing more
  dynamism) is the structural question for 2026.
- **The "model is the context" trend.** Recent papers like
  [MEMENTO]({% link _posts/2026-05-18-memento-context-management.md %})
  argue that the *model itself* should manage context. If that lands,
  serving systems will need to learn how to track *model-authored*
  cache events (segment boundaries, evictions) as a first-class API.

---

## 6. References

- Video: *Why Inference is hard..* — <https://youtu.be/B18zBnjZKmc>
- Sponsor mentioned in the video: <https://zo.computer>
- Companion survey on this site:
  [Inference Architecture report]({% link _reports/inference-architecture.md %}).
- Related on this site:
  [MEMENTO trend note]({% link _posts/2026-05-18-memento-context-management.md %}),
  [ECHO trend note]({% link _posts/2026-05-18-echo-terminal-world-models.md %}).
