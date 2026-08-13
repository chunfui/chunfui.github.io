---
layout: post
title: "Three unlocks: debugging an infra incident with an AI agent, one new data source at a time"
date: 2026-08-13 04:00:00 -0700
categories: [debugging, ai-agents, infra]
---

A couple weeks ago I spent three days chasing down a nasty infra incident —
the kind where a dozen engineers are all reporting some version of "my job is
stuck" and it's genuinely unclear whether they're all hitting the same wall or
several different ones wearing a trenchcoat. I did the whole thing with
Copilot as a debugging partner, and what stuck with me afterward wasn't really
the root cause (though that was satisfying to nail down) — it was how cleanly
the investigation scaled up every time I handed it a new tool or a new data
source. I want to write that up, partly because I think it's a genuinely good
illustration of how to work with an AI agent on an open-ended technical
problem, and partly because I'm still a little pleasantly surprised by how
each new "unlock" compounded on the last one instead of just adding noise.

## The setup

The ticket started simple enough: a wave of reports across a design team, all
some flavor of "netbatch job launch delays" and "NDM library loading hangs."
Multiple engineers, multiple projects, multiple job types — PrimeTime sessions,
Fusion Compiler floorplans, ECO database loads. On the surface it read like
one incident. Underneath, I had a hunch it probably wasn't.

What I had available at the very start was frustratingly thin: a Jira MCP tool
and a Netbatch MCP tool. That's it. No storage telemetry, no dashboard access,
nothing that could tell me what the actual filers and disks were doing.

## Phase 1 — best effort with what we had

With just Jira and Netbatch access, the approach was necessarily forensic: pull
the raw job logs directly off disk for the affected users, and lean hard on
`nbstatus` to reconstruct what netbatch itself thought was happening at the
time. This got us real signal — job wall-clock time versus actual CPU time
told a clear story for a few users (sessions running for hours while
consuming seconds of CPU, a classic sign of something stuck waiting rather
than actually computing), and cross-referencing job classes against queue
wait times pointed at a specific, congested resource class.

But there was an obvious gap. Every theory that involved "the storage side is
slow" was completely unverifiable. I had zero visibility into what the actual
filer nodes were doing — no CPU data, no IOPS data, nothing. I could describe
the symptom precisely and build a plausible theory, but I couldn't independently
confirm or rule out the storage half of the story. That gap sat there,
unresolved, for about a day and a half.

## Phase 2 — the first real unlock

The turning point came from an unplanned conversation with a colleague who
knows the storage and netbatch stack far better than I do. Talking through
what I'd found so far, he pointed me at a Grafana dashboard his team runs
internally, tracking OS-class core reservation trends pool-wide. That single
conversation led to two things happening in quick succession: getting a new
storage-monitoring tool installed, and getting real access to that dashboard.

Once I had that dashboard, the investigation took a real step forward. I
could query the exact reserved-versus-total core percentage for the specific
OS/class combination one of the affected jobs was waiting on, over the exact
incident window — not a live snapshot, but the actual historical trend. The
number came back unambiguous: that pool was sitting at roughly 90% average and
98% peak reservation for the entire window the job was stuck. For a job
spawning over a dozen distributed workers into an already-saturated,
comparatively small pool, that's about as clean an explanation for "workers
never connected" as you could ask for.

That was a genuinely good moment — not because the number was dramatic, but
because it converted a plausible theory into a confirmed one, with a real,
independently-sourced dataset behind it.

<pre class="mermaid">
flowchart LR
    subgraph P1["Phase 1 - Best effort"]
        A1["Only Jira + Netbatch<br/>available"]
        A2["Raw job logs +<br/>nbstatus queries"]
        A3["Plausible theories,<br/>storage side invisible"]
        A1 --> A2 --> A3
    end

    subgraph P2["Phase 2 - First unlock"]
        B1["Conversation with a<br/>storage/netbatch SME"]
        B2["New monitoring tool<br/>installed"]
        B3["Cross-checked his<br/>shared dashboard"]
        B4["CONFIRMED: dispatch<br/>pool at 90%+ reserved<br/>during the incident"]
        B1 --> B2 --> B3 --> B4
    end

    subgraph P3["Phase 3 - Full confirmation"]
        C1["Personal monitoring<br/>access granted"]
        C2["Pulled raw controller<br/>CPU/IOPS data for a<br/>previously invisible node"]
        C3["CONFIRMED: node pinned<br/>near 100% CPU for the<br/>entire incident window"]
        C4["Root-cause synthesis<br/>posted back to the team"]
        C1 --> C2 --> C3 --> C4
    end

    P1 --> P2 --> P3
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

## Phase 3 — closing the storage gap for good

The dispatch-congestion finding explained one user's stall precisely, but the
broader "storage is slow" reports from other engineers were still sitting
there, unconfirmed. The blind spot was specific and stubborn: one particular
filer node had zero telemetry in the monitoring tool I'd just gotten access
to. Not low activity — genuinely no data at all, like the node didn't exist
from a monitoring standpoint.

The following day, I got my own access to the underlying metrics platform
directly. First attempt at pulling data for that same invisible node still
came up empty — same blind spot, independently reproduced through a second
tool. It was only once I exported the raw CSVs scoped specifically to that
node, rather than relying on a multi-node dashboard view, that real data
finally showed up.

And once it did, it was unambiguous. Overall, that node was running at close
to 66% average CPU with regular spikes to full saturation — already a busy
node. But narrowed down to the exact incident window one user reported, CPU
utilization was pinned between 90% and 100% for essentially the entire
multi-hour stretch. Not a brief spike — sustained, near-total saturation for
hours.

That was the second real unlock, and it landed at almost the same time a
teammate independently reported having already rebalanced that exact node's
storage shelves, based purely on his own suspicion that something was off.
His fix and my data lined up perfectly, without either of us having seen the
other's evidence first.

## What the investigation actually turned up

Putting it all together, this incident wasn't one root cause — it was three,
stacking on top of each other and getting reported as a single symptom:

1. **A genuinely overloaded storage node**, confirmed independently through
   two different monitoring tools once I had access to the right one, and
   independently corroborated by a colleague's own remediation.
2. **Dispatch-level queue congestion** on a specific, comparatively small OS
   class — unrelated to storage, but producing an outwardly identical symptom
   ("my job is stuck") for a different user.
3. **A tool-version incompatibility** on a newer OS build, unrelated to
   either of the above, affecting a third set of users.

None of these were guesses. Each one had hard data behind it by the time I
wrote it up, and each one needed a different data source to confirm.

## Why I'm writing this up

The part I actually want to highlight isn't the root cause — it's the shape
of the investigation. At every stage, the moment I got a new tool or a new
piece of access, Copilot could immediately put it to productive use against
the *same open questions* from the previous phase, rather than starting over.
The dispatch-congestion theory from phase one got directly tested the moment
the Grafana dashboard became available. The storage blind spot from phase two
got directly closed the moment Graphite access came through. Nothing had to
be re-explained or re-investigated from scratch — the context carried
forward, and each new data source slotted into a specific, already-identified
gap.

That's the part that actually changed how I think about working with an AI
agent on an ambiguous technical problem: it's not that the agent solved
anything I couldn't have solved myself eventually. It's that the overhead of
picking the investigation back up after each new tool became basically zero.
Every new unlock compounded instead of requiring a restart. For a
multi-day, multi-cause incident like this one, that compounding is the whole
difference between a clean root-cause writeup and a pile of half-finished
theories.
