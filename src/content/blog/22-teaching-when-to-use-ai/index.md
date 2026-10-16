---
title: "Teaching when to use AI"
summary: "Across three sessions, an inconsistent CVE research prompt became a lesson in repeatability, vulnerability triage, and designing a Python workflow around evidence."
date: "Oct 16 2026"
tags: ["AI", "Python", "Automation", "Teaching", "Security"]
draft: false
---

Recently, teammates came to me with a problem in Stylus, our web-based AI chatbot at Citi. They had a preset that took a list of more than twenty CVEs and researched which might allow remote code execution. Repeated runs produced wildly different results.

The issues went beyond different wording. The output could omit CVEs, classify the same CVE differently on another run, or leave the team unsure whether the full list had been checked. They now had to investigate the disagreement as well as the vulnerabilities.

In [Helping Non Engineers Ship Their First AI Assisted Automation](/blog/21-enabling-first-ai-assisted-automations/), I wrote about the weekly sessions where colleagues learn to turn repetitive work into small automations. This problem gave us a useful next lesson. Becoming comfortable with AI also means learning to recognize when a task needs a different approach.

We worked through it over three sessions. We began with how language models produce answers, then defined what useful RCE triage actually meant, and finally examined how Python could retrieve the evidence from NVD.

## First, explain why the answers vary

On the first call, I started with the fundamentals. A language model generates text by predicting the next token, a small piece of text, from the context it has. It repeats that process to produce an answer. [That prediction process](https://developers.google.com/machine-learning/crash-course/llm) supports complex language tasks, but it does not guarantee that every item in a list will be checked correctly.

We covered temperature as a way to understand sampling. Where the setting is available, lower values generally reduce randomness in token selection; higher values allow more varied output. Turning it down does not verify the facts or guarantee identical answers. [Google's sampling documentation](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/capabilities/content-generation-parameters#temperature) explains that distinction. This gave us a model for understanding variation without assuming temperature was the specific cause of the Stylus results.

I used an Excel example. If a formula adds the same two fixed values, I expect the same result whether I run it or a friend runs it, on Monday or Friday. A macro that applies fixed rules to unchanged data should be equally predictable.

We discussed repeatability using the language of idempotence. More precisely, the Excel example describes determinism: the same inputs and rules produce the same output. Idempotence means repeating an action has the same effect as doing it once. Setting a cell to a fixed value is idempotent; adding one on every run is not. Both ideas matter when people need to rerun an automation.

We then considered when each kind of tool fits. A lookup or calculation with agreed rules calls for repeatable processing. Interpreting vague language or drafting an explanation involves judgment. An AI assistant can help with that interpretation or with writing the code, while defined Python functions handle the repeatable steps. For this fixed list, a standalone script was enough.

## Then, agree on what we are classifying

The second session addressed a harder problem than retrieving data: deciding what should count as potential remote code execution. We needed a rule people could explain before asking a script to apply it.

RCE means an attacker can cause code or commands to run on a system remotely. A description of a remote attacker reading a file or crashing a service does not establish code execution. We looked for evidence of code or command execution and used weakness categories, such as command injection, as supporting clues.

We also separated classification from priority. The [CVSS fields](https://www.first.org/cvss/v3.1/specification-document) for attack vector, privileges required, and user interaction help describe the conditions for exploitation. Network access with no required privileges or user interaction can raise concern, but those fields alone do not establish RCE. Evidence of known exploitation, recorded in [CISA's KEV catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog), adds another reason to prioritize review.

Local facts matter too. We discussed exposure, authentication, process privileges, compensating controls, and patch status. A mitigating factor may reduce practical risk in a particular environment; it does not change what the underlying vulnerability allows.

The material included an exercise to classify three short vulnerability descriptions by hand. One described a remote, unauthenticated attacker executing operating system commands through an HTTP request. The point was to identify the evidence behind the label and explain it. That reasoning became the basis for our script's checklist.

We settled on a first pass with room for uncertainty: potential RCE, needs review, or probably not RCE. The output should preserve the evidence behind each label so a person can check the decision. Missing information must not quietly become a negative result.

## Finally, make the data retrieval understandable

The third session covered APIs, URLs, and the shape of the NVD response. An API lets one program request data from another system. In our case, Python asks NVD for a CVE record and receives JSON, which it can read as nested dictionaries and lists.

We broke the request into two pieces: the base URL identifies the service, and the query parameter identifies the CVE we want. Then we traced where the description, CVSS metrics, and weakness information sit inside the response. Those details are defined in the [NVD API documentation](https://nvd.nist.gov/developers/vulnerabilities). Understanding that structure gives colleagues a way to inspect what the script is actually reading.

We also covered failed requests. A timeout, a proxy authentication problem, or a rate limit is a retrieval problem, not evidence that a CVE is safe. Starting with a short list and a delay between requests made the teaching example easier to follow and troubleshoot.

Together, we designed a Python workflow to read the CVE list, retrieve each record, extract the evidence our checklist needs, apply explicit triage rules, and write a small report for human review. It built on familiar concepts from earlier sessions, including loops, conditions, and error handling.

That design makes the processing repeatable, but it does not make the rules infallible. NVD records can also change. Reproducing a result requires the same source records and rule version, as well as the same CVE list.

Across these sessions, I wanted colleagues to be able to explain both the classification and the process that produced it. Before automating a decision, they should be able to work through an example by hand, identify the evidence, and say where they are uncertain. That gives them something concrete to test when they turn the work into code.
