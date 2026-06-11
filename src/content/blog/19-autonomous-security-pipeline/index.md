---
title: "Six Security Gates, One Autonomous Session"
summary: "Six phases, one Claude Code session. Storage deploy gap, fail-closed auth, a 50-test rules suite, Semgrep SAST, OWASP ZAP DAST, and email rate limiting for Vital Registry. What the new gates found the first time they ran."
date: "Jun 11 2026"
tags: ["Security", "CI/CD", "AI", "Firebase"]
draft: false
---

The original Vital Registry CI pipeline deployed to three environments and blocked on lint failures. It did not gate on security. Firestore rules had no automated test coverage. The tier check function that controls subscription access failed open. When the database read threw an error, it returned true. Authenticated users who lost their subscription got access if Firebase returned anything other than a clean read.

That failure mode was latent until a deliberate audit found it.

## The six phases

**Phase 266** was the smallest and the most embarrassing to discover. The Firebase deploy jobs listed `--only functions,firestore,hosting` and omitted `storage`. Storage rules deployed manually or not at all. One flag addition closed a class of rule drift bugs that had no automated signal.

**Phase 267** rewrote `verifyMinTier`. On any subscription fetch error, the original returned true. The rewrite made it fail closed. A read error now throws `permission-denied`. We extracted the pure `hasSufficientTier` function and `TIER_ORDER` map to `tierCheck.ts` and wrote 15 unit tests covering every tier boundary.

**Phase 268** added the rules test suite. Firestore rules had grown across multiple iterations with no automated coverage. We built a `rules-tests/` package with `@firebase/rules-unit-testing` and Vitest. 50 tests covering the owner/stranger/client write matrix for Firestore and the owner/size/type matrix for Storage. A blocking CI job gates all three deploy jobs. Firebase CLI refuses rule paths outside the project directory when running from a subdirectory, so emulators must run from the repo root. We encoded that constraint in the CI config.

**Phase 269** added Semgrep SAST. Three rulesets (`p/typescript`, `p/react`, `p/nodejs`) plus a local `.semgrep.yml` covering `eval()` (`CWE-95`, not covered by free tier rulesets). We suppressed one DOMPurify false positive with a `// nosemgrep:` annotation and a justification comment. The job blocks everything downstream.

**Phase 270** added OWASP ZAP baseline DAST against the live UAT environment. `zaproxy/action-baseline@v0.12.0` with Ajax spider, `fail_action: true`, and a `.zap/rules.tsv` suppressing four false positive rule classes with documented rationale. The adversarial reviewer caught a bug the executor missed. Both the ZAP action and an explicit `upload-artifact@v4` step uploaded an artifact with the same name. The collision caused the pipeline to fail. We removed the redundant step.

**Phase 271** rate limited `sendVerificationEmail` and `sendPasswordResetEmail`. A pure `isWithinRateLimit` helper in `rateLimit.ts`, 12 boundary tests, and a Firestore transaction enforcement wrapper to prevent two rapid calls from both reading a counter below the limit and both incrementing. The constants are named `RATE_LIMIT_MAX_CALLS=10` and `RATE_LIMIT_WINDOW_MS=60_000`. The `rateLimits/{key}` path blocks all client reads and writes in the Firestore rules.

## What the gates found immediately

Semgrep reported a blocking finding in `ContractTextPanel.tsx` on its first run. The security phases did not introduce it. It sits in a pre existing file and predates this session. `npm audit` surfaced a high severity vulnerability in `@grpc/grpc-js` inside the `firebase-admin` dependency tree, also present before this session started.

Both findings were latent. The pipeline found them the first time it ran.

## How the session ran

Six phases, one PRD issue, autonomous execution. Each phase used the same structure. A Sonnet executor plans in one context window and executes in a separate one. An Opus reviewer checks the diff against the acceptance criteria in fresh context. One fix loop where needed, then merge.

The ZAP bug is the honest example of why the review step earns its cost. The executor wrote a correct ZAP integration. It also added a redundant upload step that looked reasonable in isolation. A reviewer reading the full CI config caught the collision. That kind of gap is invisible to the agent that wrote it.

One gate is still open. App Check requires both of us to sign off on the architecture issue before implementation starts. That is deliberate. Not every decision should be autonomous.

---

For the agentic pipeline behind this session, see [Idea to Shipped](/blog/12-agentic-idea-to-ship/). For the three environment Firebase setup these gates protect, see [GitOps on a Firebase personal plan](/blog/14-gitops-firebase-personal-plan/).
