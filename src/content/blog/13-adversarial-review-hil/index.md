---
title: "AI Reviews AI: The Adversarial Review Agent and the Human-in-the-Loop Gate"
summary: "An Opus agent runs two hostile fix loops on Sonnet's output before a human sees it. Here's what the review agent looks for, how the fix loops work, and exactly where the human stays in control."
date: "May 26 2026"
tags: ["AI", "DevEx", "Automation", "CI/CD"]
draft: true
---

The executor that wrote the code cannot review it. Not because the model is incapable, but because the context is contaminated. When a Sonnet agent builds a feature, it has spent the entire session constructing a mental model of how the code works. Ask it to review its own diff and it reads what it meant to write, not what it wrote. Acceptance criteria that are "technically met" but point in the wrong direction get a pass. Logic gaps that were obvious design choices two hours ago get rationalized as correct.

The adversarial review agent solves this by being a completely different context. It has never seen the implementing conversation. It receives the diff, the changed files, the acceptance criteria from the GitHub issue, and the project conventions from CLAUDE.md. Nothing else. Its job is to derive understanding from the artifact alone, the way a peer reviewer would.

## What the review agent actually checks

The reviewer prompt is a structured skill file, not ad-hoc instruction. It forces output in a specific format: a verdict (APPROVE, REQUEST_CHANGES, or NEEDS_DISCUSSION), an acceptance criteria coverage table, and findings bucketed by severity.

Severity buckets are enforced in the prompt: CRITICAL for bugs, security issues, broken acceptance criteria, and data loss; SHOULD-FIX for convention violations and missed edge cases at boundaries; NIT for naming and formatting; QUESTION for things that look intentional but cannot be verified from the diff alone. If a bucket has no findings, the reviewer writes "None." Padding is explicitly forbidden in the prompt because vague "consider rethinking" notes are not findings, they are noise.

The acceptance criteria table is separate from the findings section on purpose. Code quality and goal coverage are two different questions. A diff can be clean, well-tested, and still miss what the issue actually asked for. The table forces the reviewer to grade each criterion explicitly, with a file and line number as evidence, or "not addressed" if it cannot find any.

Cross-links and schema compliance are checked the same way. If the diff touches content that references other files or routes, the reviewer checks those references exist. If the project uses a defined frontmatter schema, the reviewer checks every required field is present and no undeclared fields appear. These are mechanical checks, but they are exactly the kind of mechanical error the executor is most likely to miss.

## The fix loop

When the reviewer returns REQUEST_CHANGES, the orchestrator spawns a fixer subagent. The fixer is also a fresh context. It reads the REVIEW.md findings, the diff, and only the files mentioned in specific findings. It does not re-read the executor's planning documents. Giving the fixer the executor's notes would reintroduce exactly the bias the review step was designed to eliminate.

The fixer works through findings in priority order: CRITICAL first, then SHOULD-FIX, then NITs if they are cheap. For each finding addressed, the commit message references the finding ID so the review log stays traceable. Anything deferred or rejected as a false positive gets documented in FIX-LOG.md with a reason.

After the fixer commits, a fresh reviewer spawns. Not the same reviewer session continued, a new one with no memory of the prior round. It re-derives its understanding from the updated diff from scratch. This prevents cumulative bias: reviewers that have seen previous iterations have a tendency to close findings that are still present because they remember approving them earlier.

The loop runs at most twice. Two rounds of review plus two rounds of fixes is the cap.

## Why two iterations and not unlimited

Unlimited fix loops produce one of two outcomes: they converge quickly (in which case one or two rounds was always sufficient), or they thrash. Thrashing happens when findings conflict with each other, when a fix to one finding breaks another, or when the reviewer and fixer have different interpretations of a requirement that neither can resolve from the diff alone. More iterations do not resolve ambiguity, they amplify it.

After two passes, remaining open findings are judgment calls. They require a human to decide whether the implementation is pointing in the right direction, whether the acceptance criteria were written correctly in the first place, or whether a finding reflects a real problem in the code or a real problem in the spec. A third Opus round would produce a decision. It would not necessarily produce the right decision.

## Where the human sits

The human does not see the diff until after the fix loops complete. This is deliberate.

A common failure mode in AI-assisted review is presenting every raw draft to the human for approval. The human becomes the filter for mechanical errors: missing fields, broken links, logic gaps that a review agent would catch in seconds. Reviewer fatigue sets in quickly when the gate is positioned this way, and judgment degrades on the things that actually require judgment.

The HIL gate in this pipeline is positioned after review loops pass. By that point, mechanical errors have been caught and addressed. What reaches the human is a diff that has been reviewed by a fresh-context Opus agent and fixed by a fresh-context fixer, at least once. The human's attention goes to questions that the review agent cannot answer: is the framing right for the audience, should this feature exist in this form, does the implementation reflect what was actually intended when the issue was written.

On approval, the orchestrator merges to dev and cleans up the branch. On rejection, the human's specific findings go back to the executor as a new set of acceptance criteria, starting the cycle again.

## The failure mode this prevents

The executor rationalizing its own errors is the primary failure. The second is subtler: an executor that meets acceptance criteria technically while missing them in spirit.

A review agent that had access to the implementing conversation would inherit the executor's interpretation of what the criteria mean. A fresh-context reviewer has to read them as written. If the criteria say "the component renders correctly on mobile viewports" and the executor added a CSS class it believed was correct, the fresh reviewer checks whether the class actually produces the correct result, not whether the executor's belief was reasonable.

This is a concrete mechanism, not a philosophical point. The review agent is a skill file: a structured prompt with defined output format, specific anti-sycophancy rules, and a forced verdict. It runs in a subagent spawned by the orchestrator. It does not share memory with the executor. The fix loop architecture is defined in the pipeline, not negotiated at runtime.

For the full pipeline context, including how features move from idea through PRD to execution, see [the agentic dev workflow](/projects/agentic-dev-workflow/). The narrative companion that covers the end-to-end flow in practice is [here](/blog/12-agentic-idea-to-ship/).
