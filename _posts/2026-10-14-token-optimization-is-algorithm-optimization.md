---
layout: post
title: "Token optimization is algorithm optimization — we've just given it a new name"
date: 2026-10-14 00:00:00 +0000
categories: [ai-agents, tokens, engineering]
---

A colleague, Srinivas, sent around a thoughtful email this week asking a
question I think every team running agentic workflows at scale is going to
run into: when a user asks an agent to compare JSON and YAML files by just
writing it into an `agent.md`, that's fine once or twice. But what happens
when that same pattern needs to run a thousand times a release, across ten
releases, plus validation? Is paying the token cost every single time "the
cost of doing business," or should that markdown instruction be converted
into real code with solid test cases the moment it needs to run at
production scale?

His closing question was the one that matters: *what is the right approach
for production scripts that require predictable results and sign-off
quality?*

I want to answer it, but first I want to push back gently on the framing.
This isn't a new problem that AI invented. It's the oldest problem in
programming, wearing a new hat.

## We've been here before — just swap "tokens" for "wall-clock time"

Here's a script I wrote a while back. It reads an AGS entitlement group,
"DTP CDE Users," and checks whether each person already has a Perforce
license in our Common Design Environment. If not, it grants one.

```python
def check_perforce_license(user):
    cmd = f"p4 user -l {user}"
    return run(cmd)  # True/False based on whatever we're checking for

def grant_user_license(user):
    cmd = f"p4 user -i {user}"
    run(cmd)

user_list = ags.get_users("DTP CDE Users")
for user in user_list:
    if not check_perforce_license(user):
        grant_user_license(user)
```

Straightforward. It also took two minutes to run, because with 300 users it
shells out to `p4 user -l` 300 separate times. So I refactored it:

```python
p4_users = p4.get_users()
ags_users = ags.get_users("DTP CDE Users")

delta = ags_users - p4_users

for user in delta:
    grant_user_license(user)
```

One `p4 users` call instead of 300. The AGS list minus the P4 list gives me
exactly the set of people who need a license. That script runs in ten
seconds. Nothing about this refactor involves a token, a model, or an agent
— it's set subtraction replacing a per-item loop, something every engineer
learns early. The instinct to go from O(n) calls to O(1) is not new. What's
new is that we now have a second resource to optimize for besides time:
tokens. The thinking is identical.

## The 100,000-cell problem

A colleague from our ProdLib team described an agentic workflow to me
recently: an agent that hand-holds a `stm → gds/oas → spef` pipeline,
checking log output at each step before deciding whether to proceed. It
works great — until you realize he has 100,000 cells to run it against for
his actual daily work.

If you ask an agent to generate 100,000 individual commands like:

```
stm2gds --cell cellA --parameter_file=para.txt; stm2oas --cellA ...; oas2spef ...
```

one at a time, you're burning a huge number of tokens just generating text
that's 99% identical across every invocation. My suggestion was simple: tell
the agent explicitly to write a loop, or give it a wrapper script to call
once.

```
foreach cell in cells:
    cmd = "stm2gds --cell " . cell . " --parameter_file=para.txt; stm2oas " . cell . " --parameter_file=para.txt; oas2spef --some_option1 --some_option2"
```

That turns "generate 100,000 near-identical command strings" into "generate
one loop." Running the actual vendor tools doesn't cost a single token
either way — when an agent kicks off a long-running job, it's just spawning
a child process and waiting; whether that job takes one minute or ten days
makes no difference to token spend. The tokens get burned in how much text
the agent has to *generate and read*, not in how long the underlying tool
runs.

Which brings up the second half of that problem: each step needs to check a
log file before proceeding, and those log files are huge. If you let the
agent read the entire log every single time, you're back to burning tokens
for no reason. The fix there is just as old as the first one: write a small
parser, or a skill, that greps for the specific pass/fail criteria the agent
actually needs, and tell the agent explicitly to use it instead of reading
raw stdout or the full log file. Most agents will reach for `grep` on their
own if you don't give them anything else — but spelling it out as a skill
guarantees it, rather than leaving it to chance.

So the actual guidance boils down to three things, and none of them require
deep AI expertise to explain:

- Running a long vendor tool costs nothing in tokens. The only token cost is
  generating the command.
- An agent may or may not keep watching stdout depending on how you've
  instructed it — tell it not to, since you already have a log file per run.
- Give it a `grep`-based pass/fail check instead of letting it read a
  2,000-line log from top to bottom.

That's a five-minute conversation, or a short internal doc, not a research
project.

## Where agent.md actually wins

Now back to Srinivas's JSON/YAML comparison case. If someone runs that
comparison a hundred times in one session, the agent will most likely write
a small Python script once and reuse it for the other ninety-nine — that's
not actually as wasteful as it sounds. But if I were that user, I'd go one
step further: harden the comparison into a standalone `comparison.py`, and
point the agent at it from the skill or `agent.md` file. That fixes the
comparison method once and for all, so I never have to wonder whether the
agent compared things slightly differently between runs 40 and 41. The
token cost is close to a wash either way, since the agent would write that
script regardless — the real win is *determinism*, not tokens.

It's nearly impossible to build an all-markdown, no-code workflow anyway.
Even in the most "pure agent.md" setup I can think of, the agent still
writes some throwaway Python in between steps. The question was never
"code or no code" — it's "which parts of this workflow deserve to be fixed
in code, and which parts still benefit from a little inference."

And there are cases that run the other way entirely. Take Jira ticket
triage. You can absolutely write a Python script to classify and reassign
tickets by component — but it's genuinely hard to maintain, because the
rules for "which team owns this" are fuzzy and keep shifting. Compare that
to an `agent.md` paired with a simple `component_map.csv`:

```
ICV;IC Validator is a physical verification platform used for design rule
checking, layout versus schematic validation, and manufacturing signoff in
semiconductor design...
Computing;Storage, Netbatch, machines, ION, work disk
```

One markdown file and one CSV can do a job that a hand-maintained Python
classifier would really struggle with. That's exactly the kind of task
where a little inference is worth more than a rigid rule set — and it's a
case where hardening to code would actually make the system *worse*, not
better.

## A three-step maturity model, not a binary choice

Putting all of this together, here's the progression I'd suggest to anyone
building an agentic workflow, and it's the same progression I used for our
own CDE release note system — which went from a markdown-only proof of
concept to a hardened workflow in under two weeks:

1. **Write the `agent.md` first.** Let it organize the whole workflow end to
   end — and notice that part of that workflow will usually already contain
   fixed scripts (our `stm2gds.csh` wrapper, for example). Watch your token
   usage while you do this.
2. **Harden whatever doesn't need inference.** Any step that's deterministic
   — set subtraction instead of a loop, a `grep`-based log check instead of
   reading the whole file, a fixed comparison script instead of letting the
   agent re-derive it — convert to real code with real tests. Watch your
   token usage drop.
3. **Move to an AI-application model.** A script that only calls the LLM
   endpoint at the exact point where inference is actually needed, then
   handles everything else — API calls, post-processing, reassignment — in
   plain code.

For the release note system specifically, once the markdown-driven version
proved the workflow out, the next step was `/plan` to turn the steps into a
proper script-based system, a round of architecture discussion in a
spec/architecture doc (`mattpocock-skills:grill-me` is genuinely good for
this — it interviews you on your own design decisions instead of just
nodding along), `/tdd` to bake test-driven steps into the plan, and then the
conversion of `agent.md` into `release.py`.

In practice, almost every real workflow ends up being a mix of all three
steps, not a clean progression through them: harden what can be hardened,
leave genuine inference to the agent, and keep watching the token meter the
same way you'd watch a profiler. That's the honest answer to Srinivas's
question — there isn't a single "right approach" that applies uniformly to
every production script. There's a right approach *per piece of the
workflow*, and the way you find it is the same way engineers have always
found it: measure what's expensive, and fix the part that's expensive.

The algorithmic thinking never changed. We just added a second currency to
optimize for.
