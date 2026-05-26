---
title: "Three-Environment GitOps on a Firebase Personal Plan"
summary: "Dev on feature branches, UAT on PRs, prod on main — all with branch protections, Gitleaks on every run, pinned dependency versions, and weekly scheduled CVE scans. No paid plan required."
date: "May 26 2026"
tags: ["Firebase", "CI/CD", "Security", "DevEx"]
draft: true
---

The assumption is that you need a paid plan for real branch protections. For Vital Registry, a production CRM with real customer data, I built three fully isolated environments on a Firebase personal plan. Branch protections, secrets scanning, supply-chain defence, and scheduled CVE scans — all configured as code, none of it requiring a paid GitHub tier.

The motivation was direct: two engineers shipping async to a production app used by paying clients. Sloppiness has a cost when real data is at stake.

## Three environments, three promotion gates

Each environment maps to a specific trigger condition in the GitHub Actions pipeline:

**Dev** deploys automatically on every push to a feature branch. It is disposable, fast, and exists so that incomplete work has a running target. No human approval is needed — the feedback loop stays short.

**UAT** deploys on PR open or update, before the merge. This is the gate that matters. You test against a full Firebase environment with production-equivalent Firestore rules and cloud functions, not a local emulator. If it breaks in UAT, the PR sits. Nothing merges untested.

**Prod** deploys on merge to `main`, itself PR-gated. No direct pushes to main are possible — the branch protection rules block them. Force-pushes are also blocked. The sequence enforces itself by configuration, not by trust.

## Branch protections without a paid plan

GitHub's personal plan does not mandate branch protection rules. You configure them anyway. The point is that if you cannot enforce this discipline on a personal project with no one watching, you will not enforce it in a production codebase when the team is under pressure and someone is tempted to push directly to unblock a deploy. The habit is the protection.

The ruleset is minimal: no direct push to `main`, no force-push to `main`, require a PR for every merge. That is enough to guarantee that every change to production passed through UAT first.

## Gitleaks on every run

Gitleaks runs as the first step in every workflow — dev, UAT, and prod. The scan runs before Firebase deploy commands execute. If a secret pattern is detected, the run fails and nothing deploys.

The cost is a few seconds per run. The alternative is discovering a committed credential after it has already been in the Git history. Git history is permanent. Rotating a leaked key is expensive and stressful. The scan runs every time.

## Pinned dependency versions

All package versions in `package.json` and `package-lock.json` are pinned explicitly. No floating ranges, no `^` or `~` prefixes on security-relevant packages. The npm supply-chain incidents of the past few years demonstrated the attack surface: a maintainer account gets compromised, a malicious version is published, and any project using a floating range installs the malicious package on the next `npm install`.

Pinning eliminates that class of attack. The cost is manual version bumps — which is work you should be doing deliberately anyway, not leaving to an automated range resolver.

## Weekly scheduled CVE scans

Feature deployments do not follow a weekly schedule. Zero-day CVEs do not wait for deployment windows. A vulnerability published on a Tuesday exists in the production dependency tree whether or not a deploy happened that week.

A scheduled GitHub Actions workflow runs every Monday and scans the dependency tree for known CVEs. The output surfaces anything that landed in the published vulnerability databases since the last scan. Critical and high severity findings block the next deploy by design — the finding is visible before the next feature branch opens, not after it merges.

## Firestore userId as root-level collection

The Firestore schema puts `userId` at the root of every collection path. Every document lives under `/users/{userId}/...`. Data isolation is a property of the data model.

This is different from relying on middleware or application-layer access control to enforce who can read what. Middleware can have bugs. Auth logic can be misconfigured. A data model that structurally separates users by collection path cannot accidentally serve one user's data to another — the document paths make ownership visible to anyone reading the schema. No implicit trust in application logic is required.

## The actual thesis

Shift-left security is a practice, not a checklist item. Moving Gitleaks before the deploy, running CVE scans on a schedule independent of deployments, pinning versions before a supply-chain incident forces the issue — these decisions cost almost nothing individually. Collectively, they define what security hygiene looks like when it is a habit rather than a reaction.

Personal projects are where habits form. The pipeline built for Vital Registry on a personal plan is the same pipeline I would build for a team of ten on an enterprise plan, because the constraints do not change the logic. Secrets leak regardless of team size. Supply-chain attacks do not check your GitHub plan tier. Production data warrants production-grade controls at any scale.

---

This pipeline serves [Vital Registry](/projects/vital-registry-v2/). For production monitoring — quota headroom, weekly active users, and error rate on the same Firebase personal plan — see the companion post on [PostHog and Firebase alerting](/blog/15-posthog-firebase-monitoring/).
