---
layout: post
title: "From vibe coding to engineering: what Matt Pocock's skills changed for me"
date: 2026-10-19 00:00:00 +0000
categories: [ai-agents, engineering, workflow]
---

I started using Matt Pocock's [skills](https://github.com/mattpocock/skills)
last week, and I already don't want to go back. I gave the agent what I
thought was a modest job: restructure a repo and rework how users consume it.
Instead of producing code, it interviewed me. Forty questions later I had a
plan, a spec, fifteen tickets, and the first one implemented test-first.

That is the difference between vibe coding and engineering, and it is what I
want to write about. The project itself is not the point.

## The project, briefly

The project is an internal one. We maintain a shared repo of AI skills,
agents and tool servers that several engineering teams use through different
AI clients. The old setup gave everyone everything, targeted a single client,
and let behavior drift between runs. That is a poor fit when the skills drive
real design flows, where the same request should behave the same way tomorrow.

The new design organizes content by team, publishes it as versioned and
read-only bundles, and pins each user's workspace to one version. A workspace
only moves forward when the user explicitly asks it to. Here is the
[architecture diagram]({{ "/diagrams/cdeai-architecture.html" | relative_url }}),
including the parts that are planned but not yet built.

## The chain

The skills form a pipeline, and each one hands a better-defined artifact to the next:

1. **grill-with-docs**: the agent interviews you about the plan, one decision at a time, and records the answers in a glossary and decision records.
2. **to-spec**: turns the settled plan into a spec.
3. **to-tickets**: breaks the spec into vertical slices with explicit blockers.
4. **tdd**: implements one ticket at a time, red, green, repeat.

## Grilling finds the loose ends

My plan looked finished when I started. It wasn't. The agent kept asking
questions I had not thought to ask myself:

- When a user's workspace is set up with version 1.2, should it ever move to 1.3 on its own?
- What happens when a pinned version is purged?
- Who is allowed to push changes back, and what does everyone else get?
- What should the tool do if a user has edited a generated file?

Each of these is a decision I would otherwise have made implicitly, in the
middle of coding, usually under time pressure and without writing it down. The
answer to the first one ("never, unless the user asks") became the central
design principle of the whole project. I would not have stated it that
clearly if I hadn't been asked.

Two details made this work better than a plain Q&A:

- **One question at a time, with a recommended answer.** I could say "yes" when I agreed and spend my effort on the questions where I didn't. Several of my answers were "I'm not sure, but that seems reasonable", and that was fine, because the question still surfaced the implication.
- **It writes things down as it goes.** Terms get defined in a glossary, so "Group", "Bundle" and "Pin" mean one thing for the rest of the project. The big decision became a short decision record. Three weeks from now, a colleague (or I) can read why we chose it.

It also pushes back. When two of my answers contradicted each other, it said so, and I had to pick one. A plan that survives that is a much better plan.

## TDD changes what "done" means

Once the tickets existed, the tdd skill kept me honest in a way I normally am not.

- It agrees on **what to test before writing any code**. Here that meant a single seam: the builder's command line, tested against small fixture repos, not its internals.
- It writes **one failing test, then the smallest code that passes**, and repeats. No batch of speculative tests, no sprawling implementation.
- Tests assert independent, literal values, never a value compared against itself.

The practical result: the first ticket ended with 19 tests at the builder
seam, 83 passing overall, and a real dry run against the actual repo. It also
caught a coupling I had missed. A setup script cloned only one directory, and
would have silently broken with the new layout. A test failed, so I found out
before anyone else did.

With vibe coding I get code that looks right. With this I get code that has
been shown to work, and I know what it does not cover.

## What I would tell someone starting

- **Don't skip the grilling because the plan feels clear.** The feeling of clarity is exactly what the interview tests.
- **Answer honestly, including "I don't know".** The follow-up is where the value is.
- **Keep the tickets small and vertical.** Each one should be something you can demo.
- **Pause when you need to.** Midway through, I stopped to draw an architecture diagram for my colleagues. The glossary, spec and tickets made that easy, because the plan already existed in writing.

## The cost

It is slower at the start. Forty questions is a real time investment, and
some of them felt pedantic in the moment. I'd still make the trade every time:
the questions I answered up front are questions I did not have to answer
at 2am in the middle of a broken release.

We keep talking about making agents more autonomous. What I like about this
workflow is that it does the opposite for the part that matters. The agent
does the interviewing, the decomposing and the typing. I still make every
decision, and each one is written down.
