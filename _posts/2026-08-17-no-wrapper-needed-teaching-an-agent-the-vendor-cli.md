---
layout: post
title: "No wrapper needed: teaching an agent the vendor CLI instead of building one more script"
date: 2026-08-17 00:00:00 +0000
categories: [ai-agents, tooling, infra]
---

*Names, paths, and identifiers below are fictionalized — this describes a
real internal-tooling session but the actual project, library names, and
depot paths involved are confidential and not reproduced here.*

There's a debate that resurfaces every year or two on every infra team I've
been on: some hierarchical vendor system is genuinely painful to operate
correctly, and someone proposes "let's just build a wrapper script that does
the routine stuff for people." Someone else pushes back: "the vendor tool
already does this, people just need to learn it properly — a wrapper adds a
whole new thing we now have to support forever, and it'll never cover every
edge case anyway." Both sides are right, which is exactly why the debate
never resolves. This week gave me a third option I hadn't had before: skip
the wrapper, skip the "just learn it" mandate, and let an agent operate the
real vendor CLI directly — as long as it's told the *shape* of the problem
first.

## The system: hierarchical, version-stacked, unforgiving

The tool in question is an IP lifecycle management system — think of it as
version control for a tree of build artifacts, where a top-level "project
container" is really just a manifest of pinned references to sub-containers,
which are themselves manifests pinned to still-lower-level objects. Change
one leaf object, and — if you want that change to actually show up at the
top — you must re-pin the container that references it (bumping *that*
container's own version), and then re-pin whatever references *that*
container (bumping its version too), all the way up. Skip a level, and your
top-level "release" silently still points at the old leaf.

It personally took me months to really internalize this and stop fighting
it — specifically, the trick of always working **bottom-up**: finish and
verify the lowest-level change first, then update the one container that
references it, verify that, then update the next container up, and so on,
touching each level exactly once instead of thrashing back and forth. Once
you see it, it's obvious. Getting there wasn't.

## This week's task

We needed to stand up a variant of an existing project container — call it
`project.acme` — as its own independent object, `project.nova`, with three
real changes baked in:

1. Drop two sub-containers that were project-specific noise, not needed in
   the new variant.
2. Swap in a new library resource — a dependency that existed on disk but
   had never been registered as a first-class object in the system.
3. Fork one specific sub-tool's config off onto its own branch, because the
   new variant needed a different default than the original.

None of this is exotic by vendor-tool standards. All of it requires getting
the bottom-up ordering right, knowing the (fairly obscure) non-interactive
template syntax for the CLI's create/edit commands, and — this is the part
that actually cost real time historically — knowing which operations
silently *don't* do what you'd assume (more on that below).

## What I actually gave the agent

Not a runbook. Three things:

- "This is IPLM, the CLI is `pi`, here's roughly the object model (IP / Line
  / Version / alias)."
- "When you touch a hierarchy like this, always go bottom-up — finish and
  verify the lowest object first, then the container above it, one level at
  a time, and only touch the top-level project container once at the very
  end."
- "Show me the exact command before you run it" — for anything that
  actually mutates a shared, real object.

That's it. Everything else — the actual command syntax, the template file
format, the gotchas — the agent worked out live, from `--help` output and
from safely probing the tool itself.

<pre class="mermaid">
flowchart TD
    subgraph before["Before"]
        A1["setup.acme&#64;+release_x [&#64;16.TRUNK]"]
        A2["sub_a&#64;+alias1"]
        A3["sub_b&#64;+alias2"]
        A4["config_tool&#64;LATEST.acme  &lt;-- needs to change"]
        A5["sub_c&#64;+alias3"]
        A1 --> A2
        A1 --> A3
        A1 --> A4
        A1 --> A5
    end

    subgraph after["After (bottom-up, one prompt)"]
        B1["setup.nova&#64;HEAD.TRUNK [&#64;0]  &lt;-- new container"]
        B2["sub_a&#64;+alias1  (unchanged)"]
        B3["sub_b&#64;+alias2  (unchanged)"]
        B4["config_tool&#64;LATEST.nova  &lt;-- new branch, swapped in"]
        B5["sub_c&#64;+alias3  (unchanged)"]
        B1 --> B2
        B1 --> B3
        B1 --> B4
        B1 --> B5
    end

    before -. "one guided pass:<br/>branch leaf -> clone container -> swap into parent" .-> after
</pre>

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

## What it actually discovered, and got right

A few things stood out enough that I want to record them here, mostly
because they're exactly the kind of vendor-tool trivia that would otherwise
need to live in someone's head, a wiki page nobody reads, or a wrapper
script's source code.

**It found a safe way to explore a mutating CLI.** The create/edit commands
in this tool normally pop open your `$EDITOR` with a pre-filled template —
not something you can script blind. Instead of guessing the template format,
the agent set `EDITOR=cat` and ran the real command against a disposable
name. `cat` prints the temp file and exits without modifying it, so the tool
sees "no changes" and aborts the operation — nothing gets created, but you
get to see the *exact* real template shape, including sections a blank
`--help` page would never show you. It used the same trick against an
*existing* object later to dump its live field values before cloning it,
which is how it got a byzantine multi-line resource list copied verbatim
without me typing any of it by hand.

**It found the CLI's own sharp edges by reading `--help` output closely
instead of assuming.** The create command's `--template` flag turned out to
be mutually exclusive with passing a name directly (the name lives inside
the template file), while the *edit* command requires both a name and a
template. Small, easy to get backwards, exactly the kind of thing that used
to cost me a support ticket.

**It caught a real gotcha I half-remembered but couldn't fully recall:**
branching a new version-line onto an existing object registers the new
line's metadata immediately, but does **not** copy any of the underlying
file content — that's a separate, manual integrate-and-submit step against
the tool's underlying source-control backend. I asked the agent to check
before assuming the branch was "done," it checked with the backend's own
query tool, confirmed the new line was genuinely empty, and then proposed
(and, after I signed off, ran) the exact copy-forward step needed to
populate it — as a small, disposable, narrowly-scoped client so it touched
nothing outside the two paths involved.

**Renaming turned out to be safer than I assumed.** Partway through, I
realized I'd used the wrong naming convention on an object I'd already
created and already wired into a parent container. My instinct was "clone it
correctly, then delete the old one" — extra churn, and now there's a stale
object to clean up. The agent pointed out (and then verified live) that the
tool's object identity is UUID-based under the hood, not name-based — a
straight in-place rename immediately resolved correctly everywhere it was
already referenced, parent container included, with nothing left over to
delete.

None of these are things I taught it. They came from reading the tool's own
documentation and help text carefully, then verifying every claim against
the live system before acting on it — which is really the same discipline
I'd want from a careful human operator, just without the part where they
need six months of scar tissue first.

## The part that actually matters: it doesn't have to re-learn this

The step I think is the real unlock, more than any single command discovery,
is what happened *after*: I asked for all of this — the safe-probing trick,
the exact template formats, the branching gotcha, the rename behavior, the
bottom-up ordering rule — to be written into a plain markdown knowledge file
the agent reads before touching this tool again. Not a script, not a new
tool to maintain, not an abstraction layer that might not cover the next
person's edge case. Just documentation, in the same repo-adjacent place
other operational knowledge already lives, that turns "the agent
re-discovers this from scratch every time" into "the agent already knows
this."

That's the piece that actually resolves the old wrapper-vs-vendor-tool
debate, at least for cases like this one. A wrapper script is a
maintenance liability because it's *code* — it has to anticipate every
input, it can have bugs, and it needs an owner. A markdown file describing
"here's the object model, here's the one rule that matters (bottom-up),
here's how to avoid the two known gotchas" isn't a liability in the same
way. It's cheap to extend, cheap to correct, and it doesn't pretend to cover
cases nobody's hit yet — because for those, the agent still has the real
vendor CLI and its `--help` text available to figure it out live, the same
way it did this week.

We didn't get "use the vendor tool as-is" (too hard, historically, for
anyone but the one person who'd already paid the learning-curve tax). We
didn't get "build a wrapper" (a new thing to support, forever, that still
wouldn't have covered a rename-collision edge case nobody thought to code
for). We got a third thing: the real tool, operated by something that reads
the manual properly, checks its own assumptions against the live system
before committing to them, and writes down what it learns so the next
session starts smarter than this one did.
