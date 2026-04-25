---
title: "Building a Local RAG Knowledge Base for a Two-Person Agency"
summary: "We started CZ Dev as a two-person software agency. Three months in we had enough client contracts, SOWs, and meeting notes that 'where did we say that?' became a real question. This is what we built to answer it: LightRAG + RAG-Anything on an RTX 3090, Ollama for the models, MCP for Claude Code integration, Tailscale for sharing, and zero data sent to a third-party AI provider."
date: "Apr 24 2026"
draft: false
tags:
- LightRAG
- RAG-Anything
- Ollama
- BGE-M3
- Qwen2.5
- MCP
- Tailscale
- Self-hosted
- RAG
---

CZ Dev is the software agency I co-founded with my brother Zsombor. He's in Greece doing the React and TypeScript side, I'm in Budapest doing Python, data, and security. Three months in, we had enough client material — contracts, SOWs, meeting notes, internal pricing docs, an MSA template — that "where did we agree that with BEMER?" had become a recurring question with a slow answer (open Drive, search, find a near-match, open three docs to confirm). The fix is RAG. The interesting part is how we built it.

This post is the "why" companion to the [CZ-Dev-RAG project page](/projects/cz-dev-rag) — what shape we wanted, the choices we made and rejected, and the things that were harder than expected.

## What we wanted

A small set of constraints up front:

1. **Local.** Client contracts can't leave our machines. No OpenAI, no Anthropic, no third-party AI provider in the data path. Whatever we built had to run on hardware we own.
2. **Two-user shared.** Both Zsombor and I needed access from anywhere. Single-machine wouldn't cut it.
3. **Public repo, private data.** The repo had to be open-source so it could double as a portfolio artefact. Client documents had to stay in a gitignored folder.
4. **Cross-document.** Plain vector RAG retrieves the top-k chunks; it underperforms on questions where the answer spans multiple documents. We wanted "what did we agree on retainers across our clients?" to actually work.
5. **Boring to operate.** Backups, observability, CI from day one. Not a weekend hack that breaks the first time we step away.

These constraints picked the stack for us, mostly.

## Picking the framework

Plain vector RAG (Chroma + a reranker) handles "find the chunk that answers this question" well and "synthesize across three contracts" badly. Microsoft GraphRAG handles cross-document well and is heavier than what a two-person agency needs. LlamaIndex graph mode is configurable but requires more glue.

[LightRAG](https://github.com/HKUDS/LightRAG) lands between them: dual-level (low + high) graph-augmented retrieval, batteries included, actively maintained, with a built-in web UI that ships with `lightrag-hku[api]`. [RAG-Anything](https://github.com/HKUDS/RAG-Anything) wraps it with multimodal ingestion (PDFs with tables and figures, DOCX, images via OCR) — necessary because contracts are PDFs and SOWs occasionally have screenshots embedded.

The framework choice cost is that ingestion is expensive. LightRAG runs an entity-extraction LLM call per chunk, then a relation-extraction call. Plain vector RAG only embeds. For a graph that does the cross-document reasoning we need, this is fine. For a corpus where you only want "find the chunk", LightRAG is overkill.

## Picking the models for 24 GB of VRAM

The 3090 has 24 GB. Three things need to fit: the embedding model, the extraction/generation LLM, and the reranker.

**Embedding: BGE-M3.** 568M parameters, MTEB 63.0, 8K context, multilingual. Our corpus is bilingual (English + some Hungarian for the BEMER contract pieces and our own internal notes), and BGE-M3 handles both well. Alternatives — Nomic Embed Text v2 (English-strong but weaker on Hungarian), Qwen3-Embedding-8B (heavier, would compete with the LLM for VRAM), Arctic-Embed-L (English only) — all lost on either multilingual coverage or VRAM budget.

**LLM: Qwen2.5-32B-Instruct, Q4_K_M quantization, via Ollama.** This was the hardest call. LightRAG's graph quality is the LLM's quality — the entities and relations it extracts during ingestion *are* the graph. Sub-14B models produce inconsistent entity names (the same company shows up as three different nodes), brittle relation triples, and a graph that doesn't actually help at query time.

I tested 14B briefly. Confirmed the entity brittleness reports from LightRAG users. Moved up to Qwen2.5-32B at Q4_K_M (~18 GB on disk, ~18 GB in VRAM at runtime). Throughput is ~20 tok/sec on the 3090, which means a 500-PDF dense corpus takes around 6 hours to ingest. That's acceptable — ingestion is a one-time-per-document cost.

Mixtral-8x7B and Llama-3.3-70B were considered. Mixtral runs slower than Qwen-32B dense on a 3090. Llama-3.3-70B doesn't fit, even at Q4. Cloud APIs would have been faster but conflicted with the "local-only" constraint.

**Reranker: BGE-reranker-v2-m3.** Adds ~200ms per query. Measurable relevance improvement in our Ragas eval. Runs in its own docker container; pulled in after the vector + graph retrieval, before generation.

That's the model fleet: BGE-M3 + Qwen2.5-32B + BGE-reranker-v2-m3, all on the same 3090, all called from LightRAG via Ollama's HTTP API.

## Why Tailscale is the only auth layer

Two users, both with Tailscale on every device they own. Adding application-level auth (Cognito, Clerk, an API key middleware) for two people is over-engineering — and would add a thing to maintain that doesn't increase actual security.

The reasoning: the LightRAG port (9621) is bound to `0.0.0.0` on the host, but the Windows firewall and Tailscale ACLs only allow access from devices in our two-person tailnet. From the public internet, the port doesn't exist. From inside Tailscale, it's reachable but only by the device IDs in the ACL.

The threat model that matters: a leaked Tailscale auth key. If my key leaks to a single attacker device, that device gets KB access; the rest of the world doesn't. The fix is key rotation, not application auth. The runbook documents the rotation procedure.

A future client-facing version of this would obviously need real auth. That's a v2 problem if it happens, not a v1 over-engineering.

## Why Windows + Ollama + Docker Desktop, not WSL2 or dual-boot

The 3090 lives in my main Windows 11 box because that's the box I work on every day. Three options for running the AI stack on it:

1. **Dual-boot Ubuntu** — too much friction. Switching OS for KB queries kills the "use it daily" requirement.
2. **WSL2 with GPU passthrough** — works for Linux Ollama, but the recruiter install path is "install WSL, install distro, configure GPU passthrough, then everything else" which is a long quickstart.
3. **Native Windows Ollama, everything else in Docker Desktop** — Ollama for Windows uses the 3090 directly via CUDA. Containers reach Ollama at `host.docker.internal:11434`. Docker Desktop is one install. The README quickstart is "install Ollama for Windows, install Docker Desktop, clone the repo, `docker compose up`."

We picked option 3. The cost is that MinerU (the document parser) doesn't have a clean Windows GPU path, so MinerU runs on CPU in v1. Ingestion is slower than it could be. WSL2-MinerU-GPU is a roadmap item triggered if ingestion time becomes painful.

The recruiter-runnability matters more than the marginal speed gain. Anyone can clone the repo, install two things, and have a working stack in under 30 minutes (mostly model download time).

## The MCP server is the part that surprised me

The original plan was: ingest documents through the LightRAG web UI, query through the LightRAG web UI. Done.

What actually happened: every time I needed to know what a contract said while I was coding, I'd already be in Claude Code. Switching to a browser to ask the KB and switching back broke flow. Fixing this with a copy-paste workflow ("ask Claude Code, then ask the KB, then paste the answer back") was worse.

The fix is an [MCP server](https://modelcontextprotocol.io/) that wraps LightRAG's HTTP API and exposes two tools to Claude Code:

```python
@mcp.tool()
async def query_kb(question: str, mode: str = "hybrid") -> dict:
    """Query the CZ Dev knowledge base. mode: naive | local | global | hybrid."""
    response = await httpx.post(f"{LIGHTRAG_URL}/query",
                                 json={"query": question, "mode": mode})
    return response.json()

@mcp.tool()
async def list_documents() -> list[dict]:
    """List all ingested documents in the knowledge base."""
    response = await httpx.get(f"{LIGHTRAG_URL}/documents")
    return response.json()
```

Now I can ask Claude Code "what does our MSA template say about IP assignment?" in the middle of a task and get an answer without leaving the terminal. The MCP server runs as its own docker-compose service, registered in `claude_desktop_config.json`. Works in Cursor too.

This was the part of the project I'd build first if I were doing it again. The web UI is fine for ingestion and one-off queries. The MCP integration is what makes the KB actually used during work.

## Eval and tracing aren't optional

Two things every RAG project should have from day one and almost no portfolio project does:

**Evals.** Ragas with a 20-question gold set, hand-written to cover contract lookups, cross-document reasoning, narrative summary, multilingual queries, and date arithmetic. Runs against all four LightRAG retrieval modes (`naive`, `local`, `global`, `hybrid`) so we can see which mode wins on which question type. Results auto-populate the README's eval table via a `--output-readme` flag on the eval script. Without this it's just vibes.

**Tracing.** Self-hosted Langfuse via docker-compose. Every query goes through a `trace_query` context manager that records latency, input tokens, output tokens, retrieval mode, number of retrieved chunks, and rerank-applied flag. Open Langfuse on localhost:3000, see every query, drill into spans. When a query is slow or the answer is bad, the trace shows whether it was retrieval or generation that misbehaved.

Adding either of these later would be 10× the work of adding them on day one. I built the tracing module before I built the MCP server precisely because retrofitting tracing onto a system already in use is painful.

## Things I got wrong before getting them right

**`OLLAMA_FLASH_ATTENTION`.** Ollama silently enables flash attention for BERT-class embedding models in v0.13.5 onwards. BGE-M3 is BERT-based. LightRAG's merging stage embeds long entity descriptions; the F16 cast in the flash-attn path overflows on long inputs, returns NaN, returns HTTP 500, marks the document as failed. The fix is a single environment variable on the host: `OLLAMA_FLASH_ATTENTION=false`. Finding it took an embarrassing number of debug sessions. Full writeup [here](/blog/09-bge-m3-ollama-nan-embeddings-flash-attention).

**`LLM_TIMEOUT=0` is not infinite.** I assumed (reasonably!) that setting `LLM_TIMEOUT=0` would mean "no timeout". It actually means "timeout after 0 seconds". The worker uses `asyncio.wait_for(..., timeout=LLM_TIMEOUT * 2)`, and `0 * 2 = 0`. The fix is `LLM_TIMEOUT=1800`. There are three separate timeout systems in LightRAG (httpx, LLM worker, embedding worker) and they don't share semantics — `TIMEOUT=0` *does* mean infinite for the httpx layer. Documented in LEARNINGS.md.

**WSL Ollama vs Windows Ollama.** Docker Desktop boots WSL2 silently. If Ollama's systemd service is enabled inside WSL (a default for many install paths), it bridges port 11434 from WSL to the Windows host, and `host.docker.internal` routes containers preferentially to WSL — which only has the models pulled inside WSL. You see "model not found" errors even though the right models exist in the Windows Ollama install. The fix: `wsl -- sudo systemctl disable --now ollama && wsl --shutdown`.

**Per-client workspaces.** Original plan had per-client `workspace_dir` isolation. I dropped it once scope was confirmed as internal-only. The benefit of merging is cross-client pattern surfacing — "the same Firebase session-handling pattern in BEMER and KEV Explorer" is a query the unified graph can answer; partitioned graphs can't. If we ever go client-facing, multi-tenancy is a v2 concern, not a refactor.

## What this is, what it isn't

CZ-Dev-RAG is *not* a novel framework. The novelty is in the integration: choosing the right models for a fixed VRAM budget, designing an auth perimeter that fits a two-user setup without security theatre, wiring observability and CI before you need them, and shipping a public repo that recruiters can clone and run against synthetic data without seeing a single client document.

It is also the thing I'd reach for if a client asked me to build them an internal RAG over their own contracts. The architecture would change — multi-tenancy, real auth, probably Bedrock or Azure OpenAI in place of local Ollama — but the *shape* is the same: graph-augmented retrieval, an MCP wrapper for assistant integration, a tracing layer from day one, an eval harness that's actually run.

Repo: [github.com/TamasCzaban/CZ-Dev-RAG](https://github.com/TamasCzaban/CZ-Dev-RAG). Quickstart in the README gets a fresh Windows 11 box to a queryable demo in 15 minutes plus model download time. Issues and PRs welcome. Project page with the full stack table and operational notes is [here](/projects/cz-dev-rag).
