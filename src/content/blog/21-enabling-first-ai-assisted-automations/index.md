---
title: "Helping Non Engineers Ship Their First AI Assisted Automation"
summary: "Weekly Socratic sessions helped colleagues with no coding or AI background move from basic concepts to a live vendor disclosure review automation. The group built a workflow used each week that saves about two hours of review time, and at least two attendees went on to start automations of their own."
date: "Aug 05 2026"
tags: ["AI", "Automation", "Teaching", "DevEx"]
draft: false
---

The useful milestone was not getting colleagues to finish an example script. It was watching someone with no coding background describe a repetitive task, break it into steps, and use an AI assistant to build the first version.

I have been running a one hour weekly session for colleagues who started with little or no coding or AI experience. Attendance has stayed between two and five people. The sessions began with the basics, then moved into work that people needed to complete anyway.

This follows the earlier [Finding the Front Door](/blog/16-teaching-non-engineers/) post. That post describes the first sessions and the teaching format. This is the next step: moving from a practice exercise to an automation the team uses every week.

## Start with the mental model

I use a Socratic format rather than a lecture. I ask someone to explain a small data structure or a manual workflow in their own words. We follow the answer until the next question becomes clear.

For coding basics, that means working through lists, tables, fields, and transformations before writing much code. For AI tools, it means discussing what context the tool has, what it cannot verify, and how to review its output. Someone who understands those boundaries can ask for help without handing over judgment.

I also adapted teaching and grilling workflows from Matt Pocock's public skills to fit the corporate browser agent environment available to the group. The workflows kept the sessions focused on one question at a time, visible intermediate results, and review before use.

## Build a real automation together

The first live target was a weekly review and report built from a vendor supplied disclosure CSV. The team had to review the file, apply the same checks, and prepare the same report each week.

We mapped the manual work first. What arrives? Which columns matter? Which rows need attention? What does the finished report need to contain? The group used those answers to direct the automation, rather than starting from a generic prompt or a sample data file.

The resulting workflow runs each week and saves about two hours of review time. The source data is private, so this post does not include the CSV, field names, checks, or the generated report.

That boundary mattered in the session too. The AI assistant helped generate and revise code, but the people who know the process reviewed the logic and the output. They own the decision about whether the result is safe to use.

## The signal I care about

At least two attendees have started their own automations after the sessions. That matters more than attendance or a completed workshop exercise. It means they can spot a repetitive task and turn it into something they can test and improve.

I keep the format small on purpose. A one hour session makes it easier to show up with a task from the previous week, inspect what happened, and make one useful improvement. The group can ask questions while the context still exists.

The goal is to give operations colleagues a reliable first move: describe the work, identify the repeatable steps, ask the tool for a small change, then review the result against the real task.

That first move makes automation available to people who already understand the work best.
