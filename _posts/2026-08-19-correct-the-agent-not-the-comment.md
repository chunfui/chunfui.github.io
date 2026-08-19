---
layout: post
title: "I didn't like the Jira comment my agent wrote — so I fixed the agent, not the comment"
date: 2026-08-19 00:00:00 +0000
categories: [ai-agents, jira, workflow]
---

*Names, tickets, and identifiers below are fictionalized — this describes a
real live demo during an internal AI office hour, but the actual ticket
number, project names, and library names involved are confidential and not
reproduced here.*

I offered to do a quick live demo in the second half of an AI office hour
this week — "here's how I use the Jira MCP to resolve a ticket, want to see
it in action?" I expected five, maybe ten minutes. It ran closer to thirty,
because the agent didn't just answer a question — it found a missing piece
of context, discovered a real bug, fixed it, ran the tests, hit a broken
environment along the way, worked around it, and then wrote a Jira comment I
didn't like. That last part is the piece I actually want to talk about,
because correcting it taught me more than the demo itself did.

## The setup

I picked a real open ticket from my queue — a permission-request ticket
against one of my IP-permission-control agents — and, in front of a room
full of colleagues, just asked it: "can you help resolve this ticket?"

No script. No pre-staged happy path. Just the same prompt I'd use on any
normal day.

## What it did, in order

**It asked for missing information instead of guessing.** The ticket didn't
specify the project name or the sysgroup — two of the three parameters the
underlying task actually needs. Rather than picking something plausible and
running with it, the agent asked. I filled in what the user hadn't: project
name, sysgroup. If I hadn't been sitting right there, this ticket would
correctly have stayed open, waiting on the requester, rather than silently
running against a guess.

**It caught a naming conflict I only knew from experience.** The ticket text
and the copied library name didn't match — one said one project variant, the
paperwork said another. I happened to know from memory which one was
actually correct in practice, so I told it, and it proceeded. That's the
honest state of things right now: institutional memory still occasionally
has to plug the gap between what a form says and what a form *means*. That
gap is exactly the kind of thing that should get written down somewhere the
agent can read next time, not just live in my head.

**It noticed someone had already partially done the work.** Digging into the
ticket, it found that a colleague had already created part of what was being
asked for — but the ticket itself hadn't been updated to reflect that. It
didn't just barrel ahead redundantly; it checked what was already done
against its own internal playbook for this task, found the one remaining
step that hadn't happened yet, and proposed doing just that step. That's a
small thing, but it's the difference between an agent that follows a script
and one that actually reasons about state.

**It hit a real bug, and fixed it.** The script backing one of the later
steps had a genuine bug. The agent found it, proposed a fix, and — because my
instructions tell it to always test before declaring anything done — it ran
the fix through an actual test before telling me it was resolved. Not "I
think this works now." Tested, then reported.

**It hit a broken environment, and adapted instead of stalling.** The
workspace it needed to open a PR from wasn't actually connected to a GitHub
repo — a pre-existing, unrelated environment issue that had nothing to do
with the task at hand. Instead of stopping there, it reasoned its way to a
workaround: stand up a local copy, apply and verify the fix there, and come
back to the PR/repo problem separately, later, as its own issue. That's a
very human response to "the thing you needed isn't where you expected it" —
find another way to get the actual job done first, park the infrastructure
problem for later.

<pre class="mermaid">
flowchart TD
    A["Ask agent to resolve ticket"] --> B{"Missing info?"}
    B -- yes --> C["Agent asks; I supply project + sysgroup"]
    B -- no --> D
    C --> D["Check what's already done vs playbook"]
    D --> E["Found a real bug in the backing script"]
    E --> F["Fix + run real tests before reporting done"]
    F --> G{"Repo connected?"}
    G -- no --> H["Work around it: local copy, test, defer repo issue"]
    G -- yes --> I["Open PR"]
    H --> J["Draft Jira comment"]
    I --> J
    J --> K{"Comment leaks too much internal detail?"}
    K -- yes --> L["I correct it live"]
    K -- no --> M["Post as-is"]
    L --> N["Split: internal comment vs customer-facing summary"]
    N --> O["Ask agent to self-reflect, update its own instructions"]
    O --> P["Next ticket starts smarter"]
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

## The part I actually want to talk about

Once everything was fixed and tested, the agent drafted a Jira comment
summarizing what it had done — and it put too much internal detail into a
customer-facing ticket. Implementation specifics, environment quirks, the
kind of thing that's useful for me but noise (or worse, a confusing signal)
for the requester reading it.

I didn't like it. So I said so, live, in front of the room: this ticket is
customer-facing, don't expose this much internal detail — split it into an
internal comment for the record, and a short, clean summary for the
customer.

Here's the part that's easy to miss if you've never worked this way before:
I didn't rewrite the comment myself. I told the agent what was wrong with
it, and it rewrote it. Then, at the very end of the session, I asked it to
self-reflect on the whole ticket — what went well, what I had to correct —
and update its own instruction file so that *every future ticket* defaults
to a customer-facing summary plus a separate internal comment, without me
having to say it again.

That one correction is now permanent. Not because I edited a script, but
because I had a five-minute conversation with the thing that made the
mistake, and told it to remember.

## Why this is a completely typical day for me, not a special demo

Someone in the room asked, essentially: does this get saved anywhere, or do
you have to keep re-explaining yourself every time? Fair question. The
honest answer is: it depends on how new the situation is. This particular
ticket surfaced two new lessons in one pass — the missing-parameter
follow-up question, and the internal-vs-customer-facing comment split — so I
gave two corrections, and both got written into the agent's instructions by
the end of the session. On an average day now, most tickets need zero new
instructions, because the agent has already absorbed the last dozen
corrections from tickets just like it. This one ran long precisely because
it wasn't average — a bug, a broken workspace, and a comment-tone miss all
landed in the same ticket.

That's really the whole method, and it maps almost exactly onto the four
D's from Anthropic's own short course on delegation: **delegate** a task
you'd normally do yourself and understand well, **describe** the process in
plain language instead of writing a script, apply **discernment** to what
comes back — does this actually reflect the work, is this comment
appropriate for who's reading it — and be **diligent** about correcting it
out loud when it isn't. Every correction, once captured, is one the agent
never needs again.

I've been doing this long enough now that I have a handful of these agents
sitting around, one per domain — permission control, chamber setup, storage
debugging — each one a little smarter than the last because of exactly this
loop: use it, catch what it gets wrong, tell it, ask it to remember. I never
sit down and hand-author these instruction files from scratch. Every line in
them exists because something like today's demo happened once, and I decided
it shouldn't have to happen twice.
