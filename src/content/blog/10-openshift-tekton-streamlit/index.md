---
title: "Shipping Streamlit on OpenShift: Solo-Owning a 13-Stage Tekton Pipeline"
summary: "No DevEx team, no sample repo guidance beyond a starter — just a 13-stage Tekton pipeline, Helm chart config, Artifactory publishing, and an 8m15s end-to-end runtime I built and owned solo at Citi."
date: "May 26 2026"
tags: ["Kubernetes", "CI/CD", "Streamlit", "Python", "DevEx"]
draft: true
---

The sample repo had a working Dockerfile, a `values.yaml`, and a `dev-values.yaml`. That was it. No DevEx team to call, no internal Slack channel for this specific stack. Just a working local Python app and an OpenShift cluster that needed to run it on every push to main.

This is the CI/CD story behind the [Firewall & Load Balancer Vulnerability Dashboard](/projects/citi-firewall-vulnerability-dashboard/) — internal Streamlit at Citi. No public demo, no source. Mechanisms and decisions only.

## What I was working with

OpenShift is Citi's enterprise Kubernetes platform. The deployment pipeline runs on Tekton — stages run as pods, artifacts go to Artifactory, security gates are non-negotiable. The sample repo gave me a scaffold, not a playbook. The Dockerfile assumed a flat source layout; the app had a `/src` directory. The Helm values files had placeholder config. GitHub Actions needed wiring to trigger the Lightspeed/OpenShift pipeline on PR and push to main. None of that was documented for this specific case. I inherited the scaffold and owned everything else.

## The 13-stage pipeline

The Tekton pipeline runs in this order:

1. **git-clone** — checks out the repo at the triggering commit
2. **heimdall compliance checks** — Citi's Architecture Assurance gate; validates the repo and config against internal compliance rules before anything else runs
3. **python-build** — installs dependencies, runs the test suite
4. **helm-chart-publish** — packages the Helm chart and publishes it to Artifactory, tagged with the commit hash
5. **Snyk security scan** — scans dependencies and the container image for known CVEs
6. **SonarQube code quality** — static analysis, code smell detection, coverage gate
7. **container-image-build** — builds the Docker image from the app's Dockerfile
8. **deployment-trigger** — kicks off the OpenShift deployment using the newly published Helm chart
9. **finalizer** — cleanup, status reporting, notification

End-to-end runtime on a successful run: 8m15s.

The ordering is intentional. Heimdall runs first — no point scanning code that fails compliance. Chart publishes before image build so versions stay in sync. Deployment only triggers after both security gates clear.

## Helm configuration

Two values files: `values.yaml` for shared config, `dev-values.yaml` for the dev overlay. I configured both from the sample stubs — container image reference, resource requests/limits, environment variable injection (internal endpoints, shared drive paths, credential references), route config, replica count. The dev overlay differs mainly in replicas and limits; prod-level resources in a dev namespace waste quota and slow iteration.

Chart publish happens before the image build so chart version and image tag are in sync in Artifactory. Both are tagged with the commit hash — any deployed version traces back to an exact commit.

## Debugging without a net

The first several runs failed. Each time, I stripped config back to the minimum that let the pipeline initialise, confirmed that worked, then added one thing back. It's slower than reading a runbook — but there was no runbook. Failure modes were spread across stages: Dockerfile path assumptions, Helm values referencing uninjected environment variables, the GitHub Actions trigger not passing the right context to the Lightspeed webhook. Each required a pipeline run to surface, so every debugging cycle was 8 minutes even when the change was one line.

The persistent failure was the test stage. pytest couldn't resolve module imports inside the container — `/src` layout, rootdir anchoring incorrectly inside `/tests` before collection. About 15 hours across multiple days: PYTHONPATH via Dockerfile, via Helm values, `python -m pytest`, multiple conftest arrangements. Nothing resolved it cleanly.

Pragmatic call: set the test command to `pass` in `pipeline.yaml`, let the security scanning and deployment stages run, ship. Document the debt. The [companion post on that debugging session](/blog/11-pytest-rootdir-pragmatic-ship/) covers the mechanism.

## Owning a pipeline vs consuming one

Shipping through a managed golden path means a DevEx team has pre-validated the integration points and someone picks up the phone when it breaks. End-to-end ownership is different.

In the managed case, you configure a few values, open a ticket if something breaks, and wait. Here, I was both app developer and pipeline operator. Every failure was mine to diagnose. Every decision — how Helm values were structured, how artifacts were tagged, what the test stage did — had no one else reviewing it. That forces decisions to be defensible on first principles rather than inherited from a team convention.

The upside: I understand every stage of that pipeline, not just the parts that touched my app. Heimdall's compliance model, how Tekton stages pass artifacts between pods, how Artifactory tracks chart versions versus image tags — working knowledge, not theoretical.

The vocabulary of OpenShift shifts from vanilla K8s — routes instead of ingresses, imagestreams, Tekton instead of GitHub Actions runners — but everything maps. Helm is Helm. Container builds are container builds. Artifact registries work the same way whether the brand is Artifactory or ECR.

## What I'd do differently

Three things.

Add an empty `conftest.py` at the project root from day one. That's the pytest fix — one file, no configuration, anchors pytest's rootdir correctly for a `/src` layout. It cost 15 hours to find that. The [companion post on that debugging session](/blog/11-pytest-rootdir-pragmatic-ship/) covers the mechanism.

Add structured stdout logging before shipping. The Tekton pipeline doesn't attach a logging sidecar by default — anything not on stdout disappears when the pod exits. I caught this mid-debugging when I had nothing to look at. Structured JSON to stdout means the OpenShift log aggregator captures it and you can query it later.

Commit more intentionally during the debugging phase. Each debugging cycle was 8 minutes even when I knew exactly what to change. Smaller, labelled commits per stage being tested would have made it easier to diff what changed between a failing and a passing run. Hindsight, but it's a cheap habit to build.

The 8m15s runtime is the pipeline's floor — security scanning stages are non-negotiable and take what they take. That part wasn't mine to optimise.
