---
title: "The Three Numbers That Matter: PostHog + Firebase Monitoring for a 20 User Production App"
summary: "20 active users, Firebase Spark plan hard quota limits, and the rule: know your quota headroom, WAU, and error rate before your users do. PostHog event instrumentation and Firebase budget alerts keep a small app alive."
date: "May 26 2026"
tags: ["Firebase", "Python", "Automation", "DevEx"]
draft: false
---

Twenty users is not a lot. It is also twenty real people whose rental contracts, device assignments, and payment records live in the database. When the app goes down, they call.

[Vital Registry](/projects/vital-registry-v2/) runs on Firebase Spark plan — the free tier. The Spark plan is generous for a 20-user CRM. It is also unforgiving: when you hit a hard quota limit on cloud function invocations, Firestore reads, or CPU minutes, the app does not throttle. It stops, for the rest of the billing cycle. No burst allowance, no grace window. The quota resets next month.

That constraint reshaped how I think about observability. The goal is not a rich dashboard. The goal is to know three numbers before your users notice anything wrong.

## The three numbers

**Quota headroom** is the distance between current usage and the limit that kills the app. I configured Firebase budget alerts at thresholds below each hard limit: not at 90%, but earlier, at a point where there is still time to investigate. The alerts fire to email. They are not informational; they are a signal to act before the billing cycle ends.

**Weekly active users** is the retention signal. For 20 users, WAU does not need statistical significance — it needs to be visible. PostHog tracks this through event instrumentation on the key actions: device check-out, rental contract creation, payment recording. These are not vanity pageview events. They are the actions that confirm the app is being used for its purpose. If WAU drops from 18 to 11 week-over-week, that is a retention problem before any user has said a word.

**Error rate** is the reliability signal. Unhandled exceptions in cloud functions disappear undetected unless you capture them. I instrument uncaught exceptions from cloud functions as PostHog events with the function name and error type as properties. A spike in error rate on the contract-creation function before a spike in user complaints is the gap observability is supposed to close.

## Why these three

Spark plan imposes a useful constraint: every Firestore read and function call counts against the quota, so over-instrumenting is self-defeating. The discipline of picking three numbers forces a decision about what matters. Quota headroom covers infrastructure health. WAU covers product value. Error rate covers silent failures. These three span different failure modes with minimal overlap. A production incident that does not surface in at least one of them is either too obscure to matter or the instrumentation is wrong.

## PostHog setup

PostHog's free tier covers this. The React frontend initialises the client and captures events on meaningful actions: `posthog.capture('contract_created', { user_id: uid, device_id: id })`. I do not instrument every click — only the actions that represent genuine engagement with the core product. The PostHog dashboard has one insight: a WAU trend over 90 days. That is the full workflow. No funnels, no cohorts — the user base is too small for those to be meaningful.

## Firebase budget alerts

Alert configuration lives in Google Cloud Console's Billing section, not in the Firebase Console. I set two alert levels per critical quota category: one at a moderate threshold to flag elevated usage, and a second closer to the limit to demand action. Cloud function invocations are the easiest to spike: a retry bug can exhaust the counter fast. Firestore reads and writes are secondary; the access patterns are predictable. CPU minutes are a backstop. All three have tiered alerts.

This sits alongside the GitOps and branch-protection setup described in the [companion security post](/blog/14-gitops-firebase-personal-plan/). The pipeline controls what gets deployed; the alerts control whether what is deployed is behaving within the infrastructure's limits.

## Small scale, real stakes

Production-grade is not a function of user count. Vital Registry's users manage real rental contracts with real money attached. A device that goes untracked because the app was down creates a real problem for a real business.

The setup took less than a day: PostHog instrumentation across key actions is roughly 15 lines of client code, and Firebase budget alerts are a handful of form fields. The ongoing cost is zero. The ongoing time cost is a dashboard check once a week.

The three numbers are not a substitute for deeper observability if the app grows. They are the minimum viable monitoring for a production app where three specific failure modes can hurt real users. Knowing them before your users do is what monitoring is for.
