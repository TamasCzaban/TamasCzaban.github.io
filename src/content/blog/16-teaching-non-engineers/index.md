---
title: "Finding the Front Door: Teaching Non Engineers Their First Automation"
summary: "A Citi cohort had AI tools but no entry point. Socratic sessions from first principles (tokens, models, prompting) and a first project: a Pandas Excel automation replacing a manual weekly report. The blocker is never the tech."
date: "May 26 2026"
tags: ["Python", "Automation", "DevEx", "AI"]
draft: false
---

At Citi, several colleagues had access to the internal AI and developer tooling. GitHub Copilot, internal LLM interfaces, the full stack. Nobody had told them they couldn't use it. The problem was they didn't know what to type first.

That's the front door problem. The tools exist. The permissions are granted. But without an entry point, a blank prompt box is just a blank prompt box.

## First things first: what is a token

I didn't start with code. I started with questions.

What is an AI? What is a model? What is a token? These sound like orientation questions, but they're the actual foundation. If your mental model of an LLM is "smart search bar," you'll ask it questions that don't fit the shape of what it does well. You'll feel confused when it gets something wrong. You'll treat it as a black box that either works or doesn't.

Walking through how a model processes input, what a token represents, why context length matters, why the same prompt worded differently returns different outputs: that twenty-minute conversation changes how someone interacts with these tools for the rest of their time using them. It's not curriculum. It's calibration.

The Socratic format matters here. I didn't lecture. I asked questions and waited. "What do you think happens when you send a message to the model?" Let them answer. Let them be wrong. Build from there. People retain the mental model they constructed themselves far better than the one someone handed them.

## The right first project

Once the model made sense, the next decision was what to build.

The right first project is the thing they hate doing every week.

For this cohort, that was a weekly report. Every week, the same steps: open a set of Excel files, pull specific columns, aggregate, format, copy the output into a template. The whole process took about an hour. Nobody liked it. Everyone knew it was mechanical. And it was the perfect candidate.

The first project was a Pandas script that read from their source files, did the same transformations they were doing by hand, and wrote the output in the format the report required.

```python
import pandas as pd
from pathlib import Path

source_dir = Path("weekly_data")
output_path = Path("weekly_report.xlsx")

frames = []
for f in source_dir.glob("*.xlsx"):
    df = pd.read_excel(f, usecols=["Region", "Asset", "Status", "Count"])
    frames.append(df)

combined = pd.concat(frames, ignore_index=True)
summary = combined.groupby(["Region", "Status"])["Count"].sum().reset_index()
summary.to_excel(output_path, index=False)
```

That's it. Thirty seconds instead of an hour. The first time it runs correctly, the skepticism drops. Not an abstract promise: they watched it happen. That's the moment the door opens.

## The walkthrough, not the answer

The teaching mechanism that worked here was the same Socratic approach from the first session. Rather than writing the script for them, I sat with them and asked questions about their own process. "Walk me through what you do first. What file do you open? What do you look for in that file?" The automation structure came out of describing what they already did manually.

Once the structure was visible, the code followed. They weren't learning Python in the abstract. They were writing down what they already knew how to do, in a different notation.

The distinction matters for ownership. When someone watches you write code, they have a script. When they trace the logic from their own description of the task, they have understanding. And understanding is what lets them modify it six weeks later when the source file format changes.

## What this looks like as an IC skill

This wasn't a formal training program. There was no L&D involvement, no slide decks, no certification track. It was a team lead pattern: identify the repetition in a colleague's work, sit down with them, build the first version together, hand it off.

Developer enablement at the IC level looks like this. You don't need a manager role to raise the floor of your team's capabilities. You need the automation instinct, the patience to ask questions instead of providing answers, and the judgment to pick the right first project. Something with a visible payoff. Something they care about. Something small enough to ship in one session.

The same instinct that drove the BlackRock reporting automation, and every other repeating-step project, applies here. Every manual step that repeats more than twice is a scripting candidate. The only difference in an enablement context is that you're also teaching someone else to see it the same way.

## The actual blocker

After this cohort shipped their first automation, they kept going. They modified the script when the format changed. They started asking about the next repetitive task.

The technology was never the blocker. A blank prompt box and a blank Python file look the same to someone who doesn't have an entry point. The Socratic sessions provided the mental model. The Pandas project provided the proof. Together, they opened a front door that had been there the whole time.
