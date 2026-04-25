---
title: "Hunting NaN Embeddings: BGE-M3 on Ollama and the F16 Flash-Attention Trap"
summary: "Halfway through building CZ-Dev-RAG I hit a bug that took two days and a dozen wrong theories to track down: BGE-M3 returning NaN embeddings to LightRAG, but only during the merging stage, but only on long inputs, and not reproducible from a curl one-liner. The cause turned out to be a quietly-enabled flash-attention path in Ollama that overflows F16 on dense BERT inputs. Here's the debug trail and what actually fixed it."
date: "Apr 24 2026"
draft: false
tags:
- LightRAG
- Ollama
- BGE-M3
- Debugging
- RAG
- llama.cpp
- Flash Attention
---

I was halfway through building [CZ-Dev-RAG](/projects/cz-dev-rag) when ingestion started failing in a way that made no sense. Documents would extract cleanly. Entity relations would assemble. Then, during the *merging* stage, LightRAG would start logging:

```
ERROR: Ollama embed failed: HTTP 500 from /api/embed
       failed to encode response: json: unsupported value: NaN
```

LightRAG would retry three times, fail, and mark the document as failed. The extraction work was already done; the merging stage was where it died.

This is the post I wish I'd had two days earlier. It's the bug, the wrong theories, and the one-line fix that actually works.

## The setup

The embedding model is BGE-M3 served by Ollama on a Windows 11 host with an RTX 3090. LightRAG runs in Docker Desktop and reaches Ollama at `http://host.docker.internal:11434`. Qwen2.5-32B handles entity extraction and final generation. Standard LightRAG / Ollama / Windows triangle.

Ingestion has three stages:
1. **Chunking** — split the document into ~1.2K-token pieces.
2. **Extraction** — Qwen-32B extracts entities and relations per chunk; BGE-M3 embeds each chunk.
3. **Merging** — entities pulled from different chunks get deduplicated. BGE-M3 embeds the *merged* `entity_name + "\n" + merged_description` so the graph can compute similarity between entities.

The failure was always in stage 3. Stage 2 always succeeded. Documents that were short enough that stage 3 happened in one batch sometimes succeeded; longer documents always failed.

## Wrong theory 1: GPU contention

First instinct: Qwen-32B is fighting BGE-M3 for VRAM, BGE-M3 is OOMing silently, and the OOM is corrupting the output to NaN. The merging stage runs both models together, the extraction stage doesn't.

I lowered LightRAG's concurrency:

```env
MAX_ASYNC=1
EMBEDDING_FUNC_MAX_ASYNC=1
```

Now only one document at a time, only one embedding call at a time, no GPU contention. NaN still hit.

Wrong theory.

## Wrong theory 2: input length

Maybe BGE-M3 has a max token limit and merging concatenates entity descriptions long enough to blow past it. BGE-M3 advertises 8K context; surely we weren't anywhere near that.

I added input-length logging to the LightRAG embedding wrapper. The merging-stage inputs were 800 to 2500 characters — well under 8K tokens. Still NaN.

Wrong theory, or at least incomplete. Length matters, but not in the way I assumed.

## Wrong theory 3: Ollama version

Ollama had recently shipped 0.21.x. I was on 0.17.7. The error message looked like a serialization bug; maybe a fix had landed.

Upgraded to 0.21.2. NaN still hit. Same error.

Searched the Ollama issue tracker for "NaN embed" and found [ollama#13572](https://github.com/ollama/ollama/issues/13572). Six other people reporting the same thing. Some were on 0.13, some on 0.21. The bug wasn't version-specific.

That issue had the answer. Reading it more carefully than the first skim was the moment things started making sense.

## What's actually happening

Ollama serves embeddings through `llama.cpp`. In `llama.cpp/src/llama-graph.cpp:1431-1437`, the K and V tensors are cast from F32 to F16 before being passed to `ggml_flash_attn_ext`. F16's max representable value is ~65504. For longer or denser inputs, the cast overflows to `Inf`, the softmax produces `NaN`, and the embedding response contains `NaN` values that JSON can't serialize.

Flash attention should not be running on a BERT model in the first place — BGE-M3 is BERT-based and the BERT path predates the F16 flash-attn optimization. But Ollama v0.13.5 added `"bert"` to the auto-flash-attention list in `fs/ggml/ggml.go`. Since then, BGE-M3 has had flash attention silently enabled by default. The bug is in the llama.cpp path it activates.

This explains everything I'd been seeing:

- **Why merging triggers it but extraction doesn't.** Extraction embeds 1.2K-token chunks — short enough that the F16 dynamic range holds. Merging embeds `entity_name + "\n" + merged_description` — hundreds to thousands of characters of dense text — long enough to overflow F16 in the flash-attn matmul.
- **Why a curl one-liner works.** Testing with `curl -d '{"input": "Confidentiality"}'` (a single short string) doesn't reproduce. The same word inside LightRAG, with its merged description appended, fails.
- **Why concurrency reduction didn't help.** Concurrency wasn't the problem. F16 overflow was the problem.
- **Why the Ollama version didn't matter.** The bug is in llama.cpp's flash-attn path, not in version-specific Ollama code.

## The fix

One environment variable on the Windows Ollama host:

```cmd
setx OLLAMA_FLASH_ATTENTION false
```

Then kill `ollama.exe` and let it restart so it picks up the new env var. Confirm with `echo %OLLAMA_FLASH_ATTENTION%` in a fresh terminal that it's `false`.

This disables flash attention globally on the Ollama process. The cost is roughly 30% extra VRAM on the LLM (Qwen-32B, in this stack). The 3090 has headroom for that with Q4_K_M quantization — runtime VRAM goes from ~18 GB to ~22 GB, leaving 2 GB for BGE-M3. Tight, but it works.

If you're VRAM-constrained, the alternative workarounds are:

1. **Switch off Ollama for embeddings.** LightRAG maintainer `danielaskdd` recommends vLLM or SGLang for production. Both have proper BERT-class support without the flash-attn problem. Heavier to set up; right answer if you can spare the engineering time.
2. **Truncate embedding inputs client-side.** [LightRAG#1870](https://github.com/HKUDS/LightRAG/issues/1870) and the upstream fix in [LightRAG#2916](https://github.com/HKUDS/LightRAG/pull/2916) progressively truncate long embedding inputs (2000 → 1000 → 500 → 200 chars) with a placeholder fallback. Patches `lightrag/llm/ollama.py` directly.
3. **Switch to `multilingual-e5-large` via Ollama.** Same 1024-dim output as BGE-M3, so no schema rebuild. But also BERT-based, so the flash-attn caveat applies — you still need `OLLAMA_FLASH_ATTENTION=false`. Worth knowing if BGE-M3 is unavailable in your registry.

## What this changes about my mental model of Ollama

I'd been treating Ollama as a thin model server — pull a model, hit `/api/embed`, get vectors. The flash-attention episode put two things in front of me I'd been ignoring:

**Ollama makes implicit configuration decisions on your behalf.** Flash attention is on by default for BERT models since v0.13.5. There's no warning, no log line, no metadata in `/api/show` that says "this model is running with flash attention". You only find out when something downstream breaks in a way that makes no sense.

**The `llama.cpp` layer underneath is closer than it looks.** Ollama's nice CLI and HTTP API hide the fact that the actual inference runs in `llama.cpp`, with all of `llama.cpp`'s footguns intact. F16 overflow in the flash-attn path is a `llama.cpp` bug; Ollama's contribution is enabling the path by default for a model class where it's not safe.

Both of these are knowable from the Ollama codebase if you go looking. Neither is in the documentation. The lesson, I think, is that "I'll use Ollama because it's simple" is a leaky abstraction. For toy projects it's fine. For something that needs to *not silently fail*, you need to know what `llama.cpp` is doing under the hood, which means reading code.

## What I'd do differently next time

1. **Read the issue tracker before lowering concurrency.** I burned a day on `MAX_ASYNC=1` configurations. The ollama#13572 thread had been open for months with the answer.
2. **Add embedding input length and content to the trace span from day one.** Langfuse was wired in by the time I was debugging this. I didn't have the embedding input shape in the spans yet — it would have made the "stage 3 only" pattern obvious immediately.
3. **Test with a synthetic merge-stage payload, not a curl one-liner.** Reproducing with the actual content shape that fails is the fastest path to a fix. My early `curl` tests passed because they didn't match the failing input pattern.

## Things I tried that did not help

For the search engine and the next person who hits this:

- Upgrading Ollama 0.17.7 → 0.21.2. Bug is in the llama.cpp flash-attn path, not version-specific Ollama code.
- `MAX_ASYNC=1` (serial document processing). NaN still hit.
- `EMBEDDING_FUNC_MAX_ASYNC=2` (lower embed concurrency). NaN still hit.
- The "GPU contention between Qwen and BGE-M3" theory. Wrong; the NaN is purely an F16 overflow regardless of contention.
- `LLM_TIMEOUT` adjustments. Different bug, different fix.

## Recommended LightRAG + Ollama BERT-embedding settings

After the flash-attn fix, these are the env vars I'd land on for a single-3090 setup with a 32B LLM and a BERT-class embedder:

```env
# On the Ollama host (Windows: setx; Linux/Mac: export)
OLLAMA_FLASH_ATTENTION=false   # the actual fix

# In LightRAG's .env
TIMEOUT=0                       # httpx infinite (note: 0 means infinite here)
LLM_TIMEOUT=1800                # 30 min — note: 0 here means immediate timeout
EMBEDDING_TIMEOUT=300           # 5 min
MAX_ASYNC=2                     # safe on a single 3090 with flash-attn off
EMBEDDING_FUNC_MAX_ASYNC=8      # LightRAG default; concurrency wasn't the cause
```

The interesting line is `OLLAMA_FLASH_ATTENTION=false`. The rest is shaped by separate constraints.

## Repo

Full operational learnings — including this one and several others I've hit since (WSL Ollama vs Windows Ollama port collisions, three independent timeout systems in LightRAG, GitHub PR auto-close on branch delete) — live in [LEARNINGS.md](https://github.com/TamasCzaban/CZ-Dev-RAG/blob/main/LEARNINGS.md) in the CZ-Dev-RAG repo. The doc grows as new gotchas surface.

If you're running BGE-M3 on Ollama and hitting NaN, the fix is one environment variable. I hope this saves someone else two days.
