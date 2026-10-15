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

Here's a script I wrote a while back (the Perforce calls below are
illustrative wrapper functions, not literal `p4` CLI flags — the point is
the shape of the loop, not the exact syntax). It reads an AGS entitlement
group, "DTP CDE Users," and checks whether each person already has a
Perforce license in our Common Design Environment. If not, it grants one.

```python
def check_perforce_license(user):
    # wraps whatever p4 lookup confirms license/group membership for `user`
    return run(f"p4 ... {user}")

def grant_user_license(user):
    # wraps whatever p4 call adds `user` to the licensed group
    run(f"p4 ... {user}")

user_list = ags.get_users("DTP CDE Users")
for user in user_list:
    if not check_perforce_license(user):
        grant_user_license(user)
```

Straightforward. It also took two minutes to run, because with 300 users it
shells out to a Perforce lookup 300 separate times. So I refactored it:

```python
p4_users = set(p4.get_users())
ags_users = set(ags.get_users("DTP CDE Users"))

delta = ags_users - p4_users

for user in delta:
    grant_user_license(user)
```

One `p4 users` call instead of 300. The AGS set minus the P4 set gives me
exactly the people who need a license. That script runs in ten seconds.
Nothing about this refactor involves a token, a model, or an agent — it's
set subtraction replacing a per-item loop, something every engineer learns
early. More precisely, what changed is the number of *external calls*
crossing a process/network boundary: 300 subprocess round-trips collapsed
into one. The instinct to go from O(n) external calls to O(1) is not new.
What's new is that we now have a second resource to optimize for besides
time: tokens. The thinking is identical.

## The 100,000-cell problem

A colleague from our ProdLib team described an agentic workflow to me
recently: an agent that hand-holds a `stm → gds/oas → spef` pipeline,
checking log output at each step before deciding whether to proceed. It
works great — until you realize he has 100,000 cells to run it against for
his actual daily work.

If you ask an agent to generate 100,000 individual commands like:

```shell
stm2gds --cell cellA --parameter_file=para.txt; stm2oas --cellA ...; oas2spef ...
```

one at a time, you're burning a huge number of tokens just generating text
that's 99% identical across every invocation — and it's worth being
specific about why that's the expensive part: those are *generation*
tokens, which on most models cost several times more than the input tokens
used to read a prompt, and don't benefit from prompt caching the way
repeated instructions do. Generating one loop instead of 100,000 near-copies
of the same command is the cheap side of that asymmetry, not just a smaller
number of lines. My suggestion was simple: tell the agent explicitly to
write a loop, or give it a wrapper script to call once — pushing the
repetition and filtering down to the worker/tool layer instead of making
the LLM re-derive it 100,000 times.

```shell
foreach cell ( $cells )
    stm2gds --cell $cell --parameter_file=para.txt
    stm2oas --cell $cell --parameter_file=para.txt
    oas2spef --some_option1 --some_option2 --cell $cell
end
```

A terminal-based agent given a wide-open shell is just as likely to assemble
its own long, fragile pipe of `find`/`awk`/`sed` instead of something this
clean, which is worse on both axes — more tokens to generate the one-off
chain, and a brittler result. The point isn't "Python vs. shell vs. tcsh";
it's that the agent should be pointed at *one* loop or wrapper, however it's
written, instead of inventing a bespoke pipeline per cell.

That turns "generate 100,000 near-identical command strings" into "generate
one loop." Running the actual vendor tools doesn't cost a single token
either way, *provided the orchestration runtime is the one waiting* —
spawning a batch job and getting notified on exit without re-invoking the
model costs nothing whether the job takes one minute or ten days. (The
caveat matters: if the agent framework instead polls by re-prompting the
model every so often to ask "is it done yet?", every one of those check-ins
does cost tokens — so the zero-cost claim depends on the wait state living
in the tool/runtime layer, not in a chat loop.) The tokens get burned in how
much text the agent has to *generate and read*, not in how long the
underlying tool runs.

Which brings up the second half of that problem: each step needs to check a
log file before proceeding, and in chip design those logs can run into
gigabytes. Letting an agent stream or read raw stdout from a ten-hour job —
or worse, tail a live process in an active reasoning loop — is a waste of
both time and tokens, and it's not how a well-built runtime should behave in
the first place. The fix is the same systems instinct as always: push the
evaluation down. Let a local wrapper or tool run the process, check the
exit code, and grep the log for the specific error signatures that matter,
then hand the agent a small structured summary — something like
`{"status": "passed", "errors": 0}` — instead of a raw terminal dump. Most
agents will reach for `grep` on their own if you give them nothing else, but
spelling it out as a skill guarantees it instead of leaving it to chance.

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
comparison a hundred times in one session, a terminal-based agent is just as
likely to scrape by with a fragile chain of `jq`, `yq`, and `diff`, parsing
whatever raw stdout comes back, as it is to write a clean reusable script —
and the fragile-shell-chain version burns extra tokens just parsing noisy
terminal output on every single run. Either way, if I were that user, I'd go
one step further: harden the comparison into a standalone, tested
`comparison.py`, and expose it to the agent as a single clean tool from the
skill or `agent.md` file. That fixes the comparison method once and for
all, so I never have to wonder whether the agent compared things slightly
differently between runs 40 and 41, and it drops the token cost of parsing
raw output to near zero. The token cost is close to a wash at best either
way — the real win is *determinism*, not tokens.

And in chip design specifically, that win is non-negotiable, not
optional polish. Sign-off in EDA depends on repeatability: if an LLM is
regenerating an ad-hoc comparison script on the fly, you've introduced
silent verification drift between release runs — the same inputs could pass
differently depending on how the model happened to phrase the comparison
that day. Hardening that logic into a fixed script isn't just about saving
tokens; it's what makes the result auditable and reproducible enough to put
your name on. That's really the decisive answer to Srinivas's question for
anything touching sign-off quality: the moment a workflow needs to be
*provably* the same every time, it has already told you it belongs in code.

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
