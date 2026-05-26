---
title: "AI Reviews AI: The Adversarial Review Agent and the Human-in-the-Loop Gate"
summary: "An Opus agent runs two hostile fix loops on Sonnet's output before a human sees it. Here's what the review agent looks for, how the fix loops work, and exactly where the human stays in control."
date: "May 26 2026"
tags: ["AI", "DevEx", "Automation", "CI/CD"]
draft: false
---

The executor that wrote the code cannot review it. Not because the model is incapable, but because the context is contaminated. A Sonnet agent that builds a feature reads its own diff the way it meant to write it, not the way it reads. Acceptance criteria that are technically met but point in the wrong direction get a pass. The executor rationalizes logic gaps as design choices.

The adversarial review agent solves this by being a separate context. It receives the diff, the changed files, the acceptance criteria from the GitHub issue, and the project conventions from CLAUDE.md. Nothing from the implementing conversation. Its job is to derive understanding from the artifact alone.

## What the review agent checks

The reviewer prompt is a structured skill file. It forces a specific output format: a verdict (APPROVE, REQUEST_CHANGES, or NEEDS_DISCUSSION), an acceptance criteria coverage table, and findings bucketed by severity.

Severity buckets are CRITICAL for bugs, security issues, and broken acceptance criteria; SHOULD-FIX for convention violations and missed edge cases; NIT for naming and formatting; QUESTION for things that look intentional but cannot be verified from the diff alone. If a bucket has no findings, the reviewer writes "None." Vague "consider rethinking" notes are not findings.

The acceptance criteria table is separate from the findings section. Code quality and goal coverage are two different questions. A diff can be clean, well-tested, and still miss what the issue asked for. The table forces grading each criterion with a file and line number as evidence, or "not addressed."

Cross-links and schema compliance follow the same pattern. If the diff touches content referencing other routes, the reviewer checks those references exist. If the project defines a frontmatter schema, every required field must be present and no undeclared fields may appear. Mechanical checks, and exactly the kind the executor is most likely to miss.

## The fix loop

On REQUEST_CHANGES, the orchestrator spawns a fixer subagent in fresh context. It reads the REVIEW.md findings, the diff, and only the files mentioned in specific findings. The executor's planning documents stay out of scope: giving the fixer those notes would reintroduce the bias the review step was designed to eliminate.

The fixer works through findings in priority order: CRITICAL first, then SHOULD-FIX, then NITs if cheap. Each addressed finding gets a commit message that references the finding ID. Anything deferred or rejected as a false positive is documented in FIX-LOG.md with a reason.

After the fixer commits, a new reviewer spawns with no memory of the prior round, re-deriving understanding from the updated diff from scratch. Reviewers that remember previous iterations tend to close findings that are still present because they recall approving them. The loop runs at most twice.

## Why two iterations and not unlimited

Unlimited fix loops thrash. Findings that conflict with each other, or where a fix to one breaks another, amplify the disagreement across iterations rather than resolving it. After two passes, remaining open findings are judgment calls: whether the implementation points in the right direction, whether the acceptance criteria were written correctly, whether a finding reflects a real problem in the code or a real problem in the spec. A third Opus round would produce a decision, not the right one.

## Where the human sits

The human does not see the diff until after the fix loops complete. This is deliberate.

Position the gate before review loops and the human becomes a filter for mechanical errors: missing fields, broken links, logic gaps a review agent catches in seconds. Judgment degrades on the things that actually require it.

After review loops pass, what reaches the human is a diff reviewed by a fresh-context Opus agent and fixed by a fresh-context fixer. The human's attention goes to questions the review agent cannot answer: is the framing right for the audience, should this feature exist in this form, does the implementation reflect what was intended when the issue was written. On approval, the orchestrator merges to dev and cleans up the branch. On rejection, the human's findings go back to the executor as a new set of acceptance criteria.

## The failure mode this prevents

The executor rationalizing its own errors is the primary failure. The second is subtler: meeting acceptance criteria technically while missing them in spirit. A review agent with access to the implementing conversation would inherit the executor's interpretation of what the criteria mean. A fresh-context reviewer reads them as written.

If the criteria say "the component renders correctly on mobile viewports" and the executor added a CSS class it believed was correct, the fresh reviewer checks whether the class produces the correct result, not whether the belief was reasonable. The review agent is a skill file: structured prompt, defined output format, anti-sycophancy rules, forced verdict. No shared memory with the executor. The fix loop architecture is defined in the pipeline, not negotiated at runtime.

For pipeline context, see [the agentic dev workflow](/projects/agentic-dev-workflow/). The narrative companion covering the end-to-end flow is [here](/blog/12-agentic-idea-to-ship/).
