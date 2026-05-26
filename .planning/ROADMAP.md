# Portfolio Content Roadmap

Initialized for PRD #1: 12 content pieces — SWE/automation/DevEx framing.

---

## Phase 01: Blog 10 — OpenShift + Tekton Streamlit Deploy
**Status:** NOT STARTED
**Branch:** feature/3-openshift-tekton-streamlit
**GitHub Issue:** #3
**Base:** main

**Goal:** Blog post 10 — first-person narrative on shipping Streamlit on OpenShift via 13-stage Tekton pipeline

**Must be TRUE when done:**
- File exists at `src/content/blog/10-openshift-tekton-streamlit/index.md`
- Frontmatter valid: title, summary, date, tags, draft: true
- All 13 stage names listed in correct order
- 8m15s runtime and Artifactory/commit-hash tagging mentioned
- Cross-links to /projects/citi-firewall-vulnerability-dashboard/ and /blog/11-pytest-rootdir-pragmatic-ship/ present
- Astro build passes

**Parent PRD:** #1
**Depends on:** None

---

## Phase 02: Blog 12 — Agentic Idea-to-Ship Overview
**Status:** NOT STARTED
**Branch:** feature/5-agentic-idea-to-ship
**GitHub Issue:** #5
**Base:** main

**Goal:** Blog post 12 — narrative companion to the agentic-dev-workflow project page

**Must be TRUE when done:**
- File exists at `src/content/blog/12-agentic-idea-to-ship/index.md`
- All pipeline stages described (grill-me → PRD → slice → GSD execute → adversarial review → HIL gate → auto-merge)
- Cross-links to /projects/agentic-dev-workflow/ and /blog/13-adversarial-review-hil/
- draft: true, build passes

**Parent PRD:** #1
**Depends on:** None

---

## Phase 03: Blog 13 — Adversarial Review + HIL Gate
**Status:** NOT STARTED
**Branch:** feature/6-adversarial-review-hil
**GitHub Issue:** #6
**Base:** main

**Goal:** Blog post 13 — adversarial-review + human-in-the-loop gate

**Must be TRUE when done:**
- File exists at `src/content/blog/13-adversarial-review-hil/index.md`
- Explains how AI-reviews-AI and where human stays in control
- Cross-links to /blog/12-agentic-idea-to-ship/ and /projects/agentic-dev-workflow/
- draft: true, build passes

**Parent PRD:** #1
**Depends on:** None

---

## Phase 04: Blog 14 — Vital Registry GitOps
**Status:** NOT STARTED
**Branch:** feature/7-gitops-firebase-personal-plan
**GitHub Issue:** #7
**Base:** main

**Goal:** Blog post 14 — 3-environment GitOps on Firebase personal plan

**Must be TRUE when done:**
- File exists at `src/content/blog/14-gitops-firebase-personal-plan/index.md`
- 3 envs described (dev/UAT/prod), branch protections, Gitleaks, pinned versions
- Cross-links to /projects/vital-registry-v2/
- draft: true, build passes

**Parent PRD:** #1
**Depends on:** None

---

## Phase 05: Blog 11 — pytest Rootdir + Pragmatic Ship
**Status:** NOT STARTED
**Branch:** feature/4-pytest-rootdir-pragmatic-ship
**GitHub Issue:** #4
**Base:** main

**Goal:** Blog post 11 — 15-hour pytest rootdir failure and pragmatic ship decision

**Must be TRUE when done:**
- File exists at `src/content/blog/11-pytest-rootdir-pragmatic-ship/index.md`
- Real root cause explained (rootdir anchoring, conftest.py placement)
- Real fix documented (empty conftest.py at root or pythonpath in pyproject.toml)
- Cross-links to /projects/citi-firewall-vulnerability-dashboard/ and /blog/10-openshift-tekton-streamlit/
- draft: true, build passes

**Parent PRD:** #1
**Depends on:** None

---

## Phase 06: Blog 15 — PostHog + Firebase Monitoring
**Status:** NOT STARTED
**Branch:** feature/8-posthog-firebase-monitoring
**GitHub Issue:** #8
**Base:** main

**Goal:** Blog post 15 — PostHog + Firebase alerting for 20-user production app

**Must be TRUE when done:**
- File exists at `src/content/blog/15-posthog-firebase-monitoring/index.md`
- 20 active users, Spark plan quota limits, three numbers described
- Cross-links to /projects/vital-registry-v2/
- draft: true, build passes

**Parent PRD:** #1
**Depends on:** None

---

## Phase 07: Blog 16 — Teaching Non-Engineers
**Status:** NOT STARTED
**Branch:** feature/9-teaching-non-engineers
**GitHub Issue:** #9
**Base:** main

**Goal:** Blog post 16 — teaching a Citi cohort their first automation

**Must be TRUE when done:**
- File exists at `src/content/blog/16-teaching-non-engineers/index.md`
- Socratic approach, Pandas Excel automation story
- "Where's the front door" thesis present
- draft: true, build passes

**Parent PRD:** #1
**Depends on:** None

---

## Phase 08: Blog 17 — OpenStack On-Call → K8s
**Status:** NOT STARTED
**Branch:** feature/10-openstack-on-call-kubernetes
**GitHub Issue:** #10
**Base:** main

**Goal:** Blog post 17 — OpenStack on-call experience mapped to Kubernetes transferability

**Must be TRUE when done:**
- File exists at `src/content/blog/17-openstack-on-call-kubernetes/index.md`
- OpenStack concepts mapped to K8s equivalents
- 95% on-time, 90% queue utilisation, RCA structure mentioned
- draft: true, build passes

**Parent PRD:** #1
**Depends on:** None

---

## Phase 09: Project Page — Agentic Dev Workflow
**Status:** NOT STARTED
**Branch:** feature/2-agentic-dev-workflow
**GitHub Issue:** #2
**Base:** main

**Goal:** New project page — agentic idea-to-ship development workflow

**Must be TRUE when done:**
- File exists at `src/content/projects/agentic-dev-workflow/index.md`
- Full pipeline described (grill-me → PRD → slice → GSD execute → adversarial review → HIL gate → auto-merge)
- demoUrl: https://www.czaban.dev
- Cross-links to /blog/12-agentic-idea-to-ship/ and /blog/13-adversarial-review-hil/
- draft: true, build passes

**Parent PRD:** #1
**Depends on:** None
