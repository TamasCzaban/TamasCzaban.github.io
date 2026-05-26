---
title: "Idea to Shipped: Building an Agentic Development Pipeline for a Two-Person Team"
summary: "How grill-me, structured PRDs, vertical slice issues, and GSD-driven execution replaced ad-hoc async coordination for a Budapest–Greece remote team — and what it took to make the pipeline reproducible."
date: "May 26 2026"
tags: ["DevEx", "AI", "Automation", "CI/CD"]
draft: true
---

The first real problem wasn't missing a feature. It was Zsombor starting work on the wrong thing.

I had a clear picture of what we were building. The context was implicit: data model decisions, scope boundaries, what "done" meant for that slice. Zsombor is in Greece. I'm in Budapest. He picked up the task, built something reasonable, and solved a different problem. No one was wrong. The context lived in my head instead of somewhere we could both read it.

That was the failure that started the pipeline.

## The coordination problem

Two engineers across two time zones, one product. The original workflow: identify a task, hop on a call to align, the other builds. The calls weren't long but they were blocking. Sync calls don't produce artefacts. You align, you hang up, and the alignment lives in memory.

The PRD was the first fix: user stories, acceptance criteria, file-level change specs, test requirements. Something either of us, or a Sonnet agent, could pick up cold and execute against.

## Stage 1: grill-me — surfacing what you don't know you don't know

The problem with writing a PRD directly is that you write the parts you already understand. The gaps stay gaps and surface as implementation surprises.

`grill-me` is a pre-implementation interviewer. Before any code gets planned, it interrogates scope boundaries, data model decisions, edge cases, interface contracts. Not questions I would think to ask myself; I'd already answered them without noticing.

The output is a Decision Summary: every unresolved question surfaced, with a recommended position on each. The first few PRDs we wrote without this were incomplete in the same way the old sync calls were: they captured what we had thought through and skipped what we hadn't.

## Stage 2: Opus PRD writer — the handoff unit

A Claude Opus agent takes the Decision Summary and produces a structured PRD: user stories, acceptance criteria per story, file-level change specs, test requirements. Calibrated for one purpose: someone who has never seen the feature discussion should be able to implement it correctly from this document alone.

That constraint is the point. If the PRD only works for someone who was in the planning conversation, it's meeting notes, not a handoff unit. Opus handles this stage because the quality of the eventual implementation correlates with the quality of the spec, and Sonnet drifts at this level of structural coherence.

## Stage 3: vertical slice issues — no horizontal layers

The PRD gets sliced into GitHub issues, each a vertical slice: complete working functionality touching every layer it needs (data, logic, UI, test). Not "implement the backend for X" and "implement the frontend for X" as separate issues.

Horizontal slices create the coordination trap we started with: "backend done, waiting on frontend" can sit for days. A vertical slice either ships or it doesn't. Progress is binary and always demo-able.

## Stage 4: GSD execution — plan and execute in separate context windows

GSD (Get Shit Done) drives feature execution. A Sonnet agent picks up a slice issue, reads the PRD, creates a PLAN.md, and executes it: file changes, atomic commits, type checks, build verification.

Planning and execution happen in separate context windows. A single agent that plans and executes in the same context rationalises its own plan: errors compound because it reads what it meant to write, not what it wrote. The executor starts fresh from the written plan and catches gaps the planner glossed over.

## Stage 5: adversarial review — fresh context, hostile framing

After execution, an Opus agent reviews the diff in a fresh context against the PRD acceptance criteria. Framing is explicit: find what's wrong, not what's working.

Two fix loops are available. If the reviewer returns findings, the executor addresses them and the reviewer re-checks. Findings remaining after two loops escalate to human review rather than auto-resolving. The review catches what the executor rationalised, and a fresh context reads what's there instead of what was meant.

## Stage 6: HIL gate and auto-merge

If review passes, a human-in-the-loop gate triggers before merge. The human's judgment applies to the ship decision, not to cleanup after a bad merge. On approval, the pipeline auto-merges to dev and cleans up the feature branch.

End-to-end: grill-me, PRD, slice issues, GSD execution, adversarial review, HIL gate, merge. Either engineer drives any stage without a sync call. AI drives execution; the human makes the ship decision.

## The vocabulary layer

Underneath everything: a ubiquitous language document and architectural decision records. Every project-specific term defined once, referenced by agents during planning and execution. Without it, agents pick terminology from context and produce inconsistent naming across features. With it, a Sonnet agent in month six uses the same entity names as the PRD from month one. ADRs are auditable history: when the data model decision from six months ago becomes relevant, you read it.

## What shipped

Vital Registry v2 (medical device rental CRM, 20 active users), KEV Explorer (EPSS/CVSS vulnerability triage), Advisory Composer (multichannel advisory delivery). All three built async. No blocking sync calls for feature execution.

Each pipeline stage exists because something broke without it. The AI executes once the decisions are written down. The ubiquitous language and ADRs are what make the output trustworthy, not the model.

Full pipeline at [/projects/agentic-dev-workflow/](/projects/agentic-dev-workflow/). Adversarial review in depth at [/blog/13-adversarial-review-hil/](/blog/13-adversarial-review-hil/). Agency context at [/projects/czdev/](/projects/czdev/).
