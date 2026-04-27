---
title: "CZ-Dev-RAG"
summary: "Local, graph-based knowledge base for a two-person software agency. LightRAG + RAG-Anything on an RTX 3090, Windows-native Ollama, BGE-M3 + Qwen2.5-32B for embeddings and entity extraction, a BGE reranker in the retrieval path, Langfuse for tracing, an MCP server so Claude Code can query the KB as a tool, and Tailscale as the only thing standing between the world and our client contracts. Public code, private data."
date: "Apr 24 2026"
draft: false
tags:
- LightRAG
- RAG-Anything
- Ollama
- BGE-M3
- Qwen2.5
- MCP
- Langfuse
- Tailscale
- RAG
- Self-hosted
repoUrl: "https://github.com/TamasCzaban/CZ-Dev-RAG"
---

CZ-Dev-RAG is the internal knowledge base for [CZ Dev](https://czaban.dev), the software agency I co-founded with my brother Zsombor. Client contracts, SOWs, meeting notes, and our own internal docs go into a graph-augmented RAG running on the RTX 3090 in my home office. Zsombor reaches the same KB from Greece over Tailscale. Nothing is sent to a third-party AI provider; nothing is exposed publicly.

The repo is public on GitHub. The client data is not — it lives in a gitignored folder on the host, and a synthetic [`examples/demo-corpus/`](https://github.com/TamasCzaban/CZ-Dev-RAG/tree/main/examples/demo-corpus) ships in its place so anyone cloning the repo can run the full stack against five fake CZ Dev documents and get real query answers.

## What it is, in one diagram

```
data/input/{client_slug}/   ──┐
                              │
examples/demo-corpus/        ─┴──► MinerU parser (CPU)
                                       │
                                       ▼
                              ┌──────────────────────┐
                              │  LightRAG + RAG-     │
                              │  Anything            │
                              │  (Docker)            │
                              └──┬────────────┬──────┘
                                 │            │
                  extract+embed  │            │ rerank
                                 ▼            ▼
                          ┌───────────┐  ┌──────────────┐
                          │  Ollama   │  │  BGE-reranker│
                          │  (Win11)  │  │  -v2-m3      │
                          │  3090     │  │  (Docker)    │
                          │  BGE-M3   │  └──────────────┘
                          │  Qwen-32B │
                          └───────────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │  nano-vectordb   │
                       │  + file graph    │
                       └──────────────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ▼                ▼                ▼
       ┌─────────────┐   ┌─────────────┐   ┌──────────────┐
       │ LightRAG    │   │ MCP server  │   │ Langfuse     │
       │ web UI      │   │ (Claude     │   │ (tracing)    │
       │ :9621       │   │  Code tool) │   │ :3000        │
       └──────┬──────┘   └──────┬──────┘   └──────────────┘
              │                 │
              └─── Tailscale ───┘
                  (tamas + zsombor devices)
```

## Stack

| Layer | Choice | Why |
|---|---|---|
| Retrieval framework | LightRAG + RAG-Anything | Dual-level (low + high) graph-augmented retrieval beats plain vector RAG on cross-document questions. RAG-Anything wraps it with multimodal ingestion (PDF tables, images, DOCX). |
| Embedding model | BGE-M3 via Ollama | MTEB 63.0, multilingual (handles English + Hungarian — CZ Dev's bilingual corpus), 8K context, 568M params. |
| Extraction + generation LLM | Qwen2.5-32B-Instruct Q4_K_M via Ollama | Sub-14B models produce inconsistent entities and break the graph. 32B Q4 fits in ~18GB on the 3090 with headroom for BGE-M3 + reranker. |
| Reranker | BGE-reranker-v2-m3 | Modern RAG setups consistently benefit from a reranking stage. ~200ms latency cost, measurable relevance gain in the Ragas eval. |
| Tracing | Self-hosted Langfuse | Per-query latency, token cost, retrieval mode, rerank flag. Custom `metrics.jsonl` would have worked but Langfuse is a stronger portfolio signal and the right primitive for production observability. |
| MCP integration | Custom Python MCP server | Exposes `query_kb(question, mode)` and `list_documents()` as tools. Lets Claude Code query the KB during client work without opening the web UI. |
| Auth perimeter | Tailscale | Two users. Adding Cognito or API-key middleware for two people is over-engineering. Tailscale ACLs restrict access to my and Zsombor's devices; LightRAG binds to `0.0.0.0:9621` behind it. |
| Host OS | Windows 11 + Docker Desktop | The 3090 lives in a Windows box. Ollama runs natively (CUDA direct), everything else runs in Docker. Containers reach Ollama at `host.docker.internal:11434`. |
| Backup | restic to Backblaze B2 | The graph is the artefact. Daily snapshots; tested restore drill in the runbook. |
| Document parsing | MinerU (CPU in v1) | Handles PDFs, scanned tables, images. WSL2 + GPU passthrough for MinerU is a roadmap item, not a v1 blocker. |
| Eval harness | Ragas + 20-question gold set | Faithfulness, answer relevancy, context precision across all four LightRAG retrieval modes (`naive`, `local`, `global`, `hybrid`). Auto-populates the README's eval table. |
| CI | GitHub Actions | Lint (ruff), typecheck (mypy), test (pytest), compose-smoke (boots the full stack and runs a healthcheck). |

## Engineering highlights

**Tailscale as the only auth layer.** This is a decision people will quibble with. The reasoning: two people, an internal tool, no public exposure surface, and the WireGuard tunnel that Tailscale wraps is itself the strongest part of the security model. Adding application auth on top would be theatrics — what stops an attacker is that the LightRAG port isn't reachable from the internet at all. ACL pinning to two device IDs means a leaked tailnet auth key would only burn the device that received it, not the whole KB.

**One unified graph, no per-client workspace isolation.** The original design had per-client `workspace_dir` directories. I dropped this once the scope was confirmed as internal-only. The benefit of merging is that cross-client patterns surface automatically — "the same Firebase session-handling pattern showed up in BEMER and KEV Explorer" is a query the unified graph can answer; partitioned graphs can't. If CZ Dev ever productizes this as a client-facing service, multi-tenancy is a v2 concern, not a refactor.

**MCP server as a first-class component, not an afterthought.** Tamas and Zsombor both use Claude Code as a daily driver. Letting Claude Code query the KB as a tool — via a small Python MCP server that wraps LightRAG's HTTP API — is dramatically higher-value than always opening the web UI. The MCP server runs as its own docker-compose service and reads from the same LightRAG stack. The README has the `claude_desktop_config.json` snippet.

**Eval harness with a 20-question gold set.** A handwritten gold set covering contract lookups, cross-document reasoning, narrative summary, multilingual queries, and date arithmetic — the five query types the demo corpus is designed to exercise. Ragas runs these against all four LightRAG retrieval modes (`naive`, `local`, `global`, `hybrid`), writes per-mode CSVs to `evals/results/`, and auto-populates the README's eval table via a small `--output-readme` flag on the eval script.

**Langfuse tracing from day one.** Every query runs through a `trace_query` context manager that records latency, input/output tokens, retrieval mode, number of retrieved chunks, and whether the reranker fired. Self-hosted Langfuse via docker-compose; no external dependency. Open the UI on `localhost:3000`, see every query, drill into individual spans.

**Compose-smoke CI.** `compose-smoke` is a CI job that boots the entire docker-compose stack on a clean ubuntu-latest runner and waits for every healthcheck to go green before exiting. Catches any compose configuration drift the moment it lands in a PR. Runs on every push to main.

## Eval results

20-question gold set, scored with Ragas (Qwen2.5-7B as the judge model, BGE-M3 for similarity), four LightRAG retrieval modes:

| Mode   | Faithfulness | Answer Relevancy | Context Precision |
|--------|-------------:|-----------------:|------------------:|
| naive  | 0.975 | 0.897 | 0.950 |
| local  | 0.992 | 0.877 | 0.950 |
| global | 0.969 | 0.867 | 0.950 |
| **hybrid** | **1.000** | 0.861 | **1.000** |

Hybrid wins on faithfulness and context precision; naive edges it on answer relevancy because the question phrasing in the gold set leans towards single-document lookups, which `naive` handles directly without the graph traversal step. The tight clustering across modes (<0.04 spread on faithfulness, <0.04 on relevancy) confirms the graph is healthy — entities are deduplicated cleanly, retrieval surfaces the right chunks regardless of mode. A noisier graph would show one mode dramatically outperforming the others.

LightRAG's `kv_store_llm_response_cache.json` makes rerunning the eval cheap. The first run takes ~30 minutes against a cold cache (each of 80 queries hits Qwen2.5-32B). Subsequent runs against the same questions complete the query stage in 5-9 seconds per mode; only the Ragas scoring pass against the 7B judge has to actually run. This makes the eval suitable for tightening prompt templates or reranker thresholds without paying the full ingest-and-extract tax each time.

v1 is shipped end-to-end: 10 phases merged to main, CI green, branch protection enforced, restore drill from B2 tested, bilingual (EN+HU) OCR smoke fixtures in place.

## Operational learnings worth flagging

- **`OLLAMA_FLASH_ATTENTION=false` on the host is mandatory** for BGE-M3 to stop returning NaN embeddings on long inputs. Ollama silently enables flash attention for BERT-class models in v0.13.5+; LightRAG's merging stage embeds long entity descriptions that overflow F16 in the flash-attn path, producing NaN, producing HTTP 500. This took an embarrassing number of debug sessions to track down. Full writeup in the [companion blog post](/blog/09-bge-m3-ollama-nan-embeddings-flash-attention).
- **Three independent timeout systems in LightRAG** — `TIMEOUT` (httpx), `LLM_TIMEOUT` (worker `asyncio.wait_for`), `EMBEDDING_TIMEOUT` (embed worker). Setting one does not affect the others. `LLM_TIMEOUT=0` does not mean "infinite" — it means "timeout immediately". Documented in `LEARNINGS.md`.
- **WSL Ollama vs Windows Ollama port collision.** Docker Desktop boots WSL2; if Ollama's systemd service is enabled inside WSL it bridges port 11434 from WSL to the Windows host, and `host.docker.internal` from containers routes preferentially to WSL — which has the wrong model list. `wsl -- sudo systemctl disable --now ollama` and a `wsl --shutdown` is the permanent fix.
- **`OLLAMA_HOST=0.0.0.0`** in the Windows registry is a footgun for Docker Compose. Docker Compose interpolates env vars from the host into containers — `LLM_BINDING_HOST=${OLLAMA_HOST}` would give containers `0.0.0.0` instead of `http://host.docker.internal:11434`. Hardcode the container-side URL directly in `docker-compose.yml`. Never interpolate `$OLLAMA_HOST`.

The full LEARNINGS.md is in the repo and grows as new gotchas surface.

## What this project demonstrates

The point of CZ-Dev-RAG is not the code — LightRAG and RAG-Anything do most of the heavy lifting. The point is the integration work around them: choosing the right models for a 24GB VRAM budget, hardening Ollama on Windows, designing the auth perimeter for a two-user setup, wiring observability and CI from day one rather than retrofitting them, and shipping a public repo that recruiters can clone and run end-to-end against synthetic data without ever seeing client material.

It's also the thing I'd reach for if a client asked me to build them an internal RAG over their contracts. The architecture would change — multi-tenancy, real auth, probably Bedrock or Azure OpenAI in place of local models — but the shape is the same. Knowing the shape from running it for ourselves first is the point.

## Open source

Repository at [github.com/TamasCzaban/CZ-Dev-RAG](https://github.com/TamasCzaban/CZ-Dev-RAG). 10 implementation phases all merged to main; CI green; the README has a 15-minute quickstart that gets a fresh Windows 11 machine to a queryable demo (model pull time excluded). Issues and PRs welcome.
