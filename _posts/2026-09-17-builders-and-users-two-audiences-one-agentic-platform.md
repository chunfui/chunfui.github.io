---
layout: post
title: "Builders and users need two different AI products, not one"
date: 2026-09-17 00:00:00 +0000
categories: [ai-agents, platform, workflow]
---

I started using VS Code back in September of last year, but I only picked up
GitHub Copilot CLI this past March. In the six months since, I've gone from
"just chatting with an assistant" to building out a small ecosystem of
MCP servers, agents, skills, and knowledge files that fine-tune my whole
Copilot CLI environment for Intel-specific work. Somewhere in that journey I
noticed something that I think matters more than any individual agent I've
built: the people who will use this platform split cleanly into two very
different audiences, and giving both the same product is a mistake.

## How I actually work — and why it's backwards from the tutorials

If you go looking for how to write a `SKILL.md` or an `agent.md`, the
tutorials are consistent: here's the frontmatter format, here's the folder
structure, write your skill definition, save it, done. I've watched those
videos. I've never once opened an editor and hand-authored one from scratch.

Here's what I actually do instead. I bring a real problem to Copilot, give it
context, and we talk it through — plan, debug, iterate — exactly like I would
with a coworker. Only *after* the task is done do I ask: "based on the debug
we just did, self-reflect on what could have gone better, and update the
relevant README, knowledge file, or agent instructions." Copilot writes the
skill file. Not me.

That's a genuinely different working style from "software development with
an AI pair programmer," and I don't think it's a small distinction. I treat
Copilot as a coworker who's capable of learning everything I know, so I can
progressively hand off more of my own job to it. My `copilot-instructions.md`
has grown out of real corrections, not upfront design — rules like "never
write log files into the home directory, Intel's quota is 80MB" exist
because I hit that wall once and told Copilot to remember it. I keep pointers
to on-demand knowledge files (`$COPILOT_HOME/knowledge/intel-ldap.md`,
`storage.md`, `netbatch.md`, and others) so Copilot only loads Intel-specific
context when a task actually needs it, instead of front-loading everything
into every session.

The loop is simple and it compounds:

<pre class="mermaid">
flowchart LR
    A["Bring a real problem"] --> B["Work it with Copilot: context, plan, debug"]
    B --> C["Task completed"]
    C --> D["Ask Copilot to self-reflect"]
    D --> E["Copilot updates README / knowledge/*.md / agent.md"]
    E --> F["Next similar task starts smarter"]
    F --> A
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, flowchart: { useMaxWidth: false }, themeVariables: { fontSize: '18px' } });
</script>

I enjoy this part of the work honestly — not just the debugging, but the act
of building the agent itself. Every time I hand a new corner case to one of
my agents, if it struggles, I nudge it: a hint, a direction, a piece of
context it didn't have. I watch it work. Then, after the fact, I ask it to
capture what it learned. The agent that shows up next time is measurably
smarter than the one I used last week — not because I rewrote its
instructions, but because it wrote them itself, from a real case, right
after living through it.

This is genuinely revolutionary for how much it lifts off my plate on almost
every task I touch. But I've come to realize it's also a style of working
that only appeals to a specific kind of person.

## Not everyone wants to be a builder

Some people are builders. They'll happily sit with an agent, guide it through
a rough patch, and enjoy watching `agent.md` get smarter with every session.
But plenty of people are just users — and that's a completely legitimate way
to want to work. They aren't looking to *explore* anything with Copilot. They
want to type a request and be handed a working result.

Here's the twist: the tool they're waiting for already exists — they just
don't know it. A user could type this into GHCP CLI today: "This is my
regression result area — the results are separated into folders named after
the flow, and each folder has a `flow_name.pass` or `flow_name.fail` file. I
want a CSV that combines all the run results, and an HTML dashboard showing
pass/fail status per block. Can you guide me through this?" — and it works,
right now, no setup beyond what's already installed.

But that's not what a user *expects* "an AI tool" to look like. They don't
picture themselves writing a paragraph like that from scratch; they picture
opening something and being asked questions: "Where is your workspace?" →
Copilot reads the folder structure → "Where are the pass/fail reports?" →
"How do you want pass/fail status represented?" — and out comes the
dashboard. Both paths get to the same place. The one-shot prompt above and a
polished wizard-style agent that asks the same questions one at a time
produce an identical result. The difference is entirely in expectation, not
capability: a builder is happy to discover the prompt that works. A user
expects Copilot to ask *them* the questions, and doesn't expect to have to
know the prompt exists in the first place.

## Two audiences, two different products

This is the part I think is easy to miss if you only talk to the builders in
the room: **builders and users need two different things shipped to them,
not one platform with an extra README.**

**For builders**, the bar is already low, and it should stay low. All they
need is VS Code and GHCP CLI — both already installed centrally under
`/p/dtp/eda` — plus one setup script to wire up their environment. That's the
entire onboarding cost. Anything more than that risks turning a
builder-friendly tool into ceremony.

**For users**, the bar needs to be zero. Not "here's how MCP config works,"
not "here's where skills live" — none of that should be visible to them at
all. They should run one command, get a fully wired agentic environment, and
just start chatting. This is exactly what I've been building with **CDE-AI**:
a single published tool (`infra/cdeai` in our CDE Argon repo) that launches
GitHub Copilot CLI from inside a live workspace, in its own terminal, with
every agent, skill, and MCP server already configured — no separate setup
step, no knowledge of the underlying wiring required.

Picture what that actually looks like in practice. A user launches CDE-AI
from their workspace and just types: *"Please run all APR flow on block DHM,
and give me an update every 30 minutes on the run status."* They don't need
to know where the APR recipes live, what commands populate a block into the
workspace, or how any of the automation is wired — the agent reads the
relevant instructions, figures it out, and gets the job done.

Or take a debugging case I think about a lot: a user submits an STA job to
Netbatch. Their last job took an hour; this one has been running for three
and shows no sign of finishing. They ask: *"I submitted an STA job and it's
still running after 3 hours — my previous job only took 1 hour. Can you check
and tell me what happened?"* Copilot pulls the job history through the
Netbatch MCP server (invisible to the user), reads the STA log, consults an
`sta-debug` skill file for the known failure patterns, and comes back with an
actual diagnosis — no manual log-diving, no knowing which MCP tool to call.

## The steering loop still exists — it's just invisible to the user

Here's the part that makes this more than a black box: users watching
Copilot "think out loud" during a session are bound to spot a place where its
method is inefficient, or where they happen to know something it doesn't.
They can steer it in plain language — *"I have an optimization recipe for STA
on this block at `/nfs/site/disks/location/recipe.txt`"* — and Copilot folds
that into the session immediately. The missing piece is making sure that
correction doesn't just live in one session and evaporate: wiring Copilot to
turn steering like that into an update to the shared knowledge base, and open
a PR back to the central repo, so the next user's session starts one step
ahead. (The mechanics of that feedback loop are in `infra/cdeai`'s own
`AGENTS_ADDENDUM.md` if you want the details — the short version is the tool
already sets up a writable clone specifically for capturing this kind of
feedback.)

So the user experience is a black box, but it's not a *closed* box. It just
means the person steering it doesn't need to know they're steering an agent
platform at all — they think they're just correcting Copilot, and that's
exactly right.

<pre class="mermaid">
flowchart TD
    A["User launches CDE-AI"] --> B["Prompts in plain language"]
    B --> C["Copilot silently wires MCP / skills / knowledge"]
    C --> D["Task executes: dashboard, STA debug, APR run, etc."]
    D --> E{"User spots something to correct?"}
    E -- yes --> F["User states the correction / gives a recipe"]
    F --> G["Session applies it immediately"]
    G --> H["Copilot updates shared knowledge base"]
    H --> I["PR back to central repo"]
    I --> J["Next user's session starts one step ahead"]
    E -- no --> K["Task done"]
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, flowchart: { useMaxWidth: false }, themeVariables: { fontSize: '18px' } });
</script>

## Where this leaves the platform strategy

I don't think the answer is to pick one audience over the other. It's to
keep serving both, deliberately, with different products:

- Keep the **DTP AI marketplace** — MCP servers, agents, skills — as the
  builder's toolkit. Low floor, high ceiling, "bring your own problem and
  we'll help you wire it up."
- Build the **all-in-one, black-box experience** — CDE-AI — for the much
  larger population of users who just want to open a prompt and get their
  job done, with zero knowledge of what's running underneath.

Both need to exist at the same time, served from the same underlying set of
agents and knowledge, because the builders are the ones actually producing
the skills and corrections that make the black box smarter every week. The
users just don't need to see any of that machinery to benefit from it.
