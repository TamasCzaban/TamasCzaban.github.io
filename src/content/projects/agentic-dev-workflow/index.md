---
title: "Agentic Dev Workflow: Idea to Ship"
summary: "A golden path CI/CD of the mind: grill-me surfaces decisions, Opus scopes a PRD, Sonnet executes vertical slices, an adversarial review agent runs fix loops, and a human in the loop gate approves before auto-merge."
date: "May 26 2026"
tags: ["DevEx", "AI", "CI/CD", "Automation"]
demoUrl: "https://www.czaban.dev"
draft: false
---

CZ Dev runs as two engineers — Tamas in Budapest, Zsombor in Finland/Greece. No standup cadence, no blocking sync on requirements, no "can you review this?" over Slack. This pipeline is how features get from idea to merged without either of us waiting on the other.

It is not a one-off script. Every stage is a skill file with defined inputs, defined outputs, and deterministic behaviour. One correct way to do the thing. A golden path.

## The pipeline

**grill-me** opens every feature. It is a pre-implementation interviewer: it asks about every unresolved architectural decision before any plan is written. Scope, data model, auth, edge cases, failure modes. The output is a Decision Summary — a structured record of what was decided and why. Nothing enters the PRD that wasn't surfaced here.

**Opus PRD writer** takes the Decision Summary and produces a structured Product Requirements Document: user stories, acceptance criteria, file-level change specifications, and test requirements. The PRD is the handoff unit. A human or an AI agent can pick it up cold and know what to build. This is what makes async work possible — the knowledge transfer happens once, in writing, not on a call.

**prd-to-issues** slices the PRD into GitHub issues. Each issue is a vertical slice: one working piece of the product end-to-end per issue. No horizontal layers, no "backend only" issues that block a frontend issue that blocks a test issue. Each slice ships on its own or not at all.

**GSD execution (Sonnet)** picks up a slice issue, reads the corresponding PRD section, writes a PLAN.md, and executes it. Atomic commits, type checks, build verification at each step. Planning and execution run in separate context windows on purpose — the executor cannot revise the plan mid-execution to rationalise what it already did. The plan is fixed before the first file changes.

**Adversarial review (Opus)** opens a fresh context and reads only the diff and the acceptance criteria. It has no memory of what the executor intended. It finds issues the executor missed because it was not there when the decisions were made. Two fix loops maximum. If findings survive two loops, the issue escalates to human review rather than pushing a marginal fix.

**HIL gate**: human reviews the diff and approves or rejects. The human's judgment covers whether the implementation is correct for the domain, not just whether it compiles and tests pass.

On approval: auto-merge to dev, branch cleanup.

## What it shipped

[Vital Registry v2](/projects/vital-registry-v2/) — the React production rewrite of the medical device CRM, built async across two time zones. [KEV Explorer](/projects/kev-explorer/) and [Advisory Composer](/projects/advisory-composer/) — both public security tools, both built through the same pipeline.

The skills themselves are internal tools. There is no public repo. The pipeline is real, and the products it shipped are live.

## Why it works

The PRD is the handoff unit because it forces the knowledge transfer to happen before work starts, not during. A ubiquitous language doc and living ADRs give the AI's output a shared vocabulary to anchor to — that document-grounding is what makes the generated code trustworthy, not the model. Separate planning and execution contexts prevent the rationalization loop where an AI agent edits its own plan to fit what it already built. And the adversarial reviewer works because fresh context is not a limitation — it is the mechanism. It sees the diff the way a code reviewer who was not in the room sees it.

---

Narrative companion: [Why we built an agentic golden path](/blog/12-agentic-idea-to-ship/)

Deep dive on the review stage: [Adversarial review and the HIL gate](/blog/13-adversarial-review-hil/)
