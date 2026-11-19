---
layout: post
title: "I promoted my AI assistant to Chief of Staff"
date: 2026-11-19 00:00:00 +0000
categories: [ai-agents, workflow, productivity]
---

I started with one agent and one terminal. A few months later I had five to ten
terminals open, each working a different domain. Today it is closer to twelve
to fifteen, because sometimes I spawn a second or third session in the same
domain to cope with the load. This post is about what that did to my head, and
the fix that has worked better than I expected. It comes down to four ideas:

1. Nami, my Chief of Staff, holds the horizontal context across every domain, and each Worker holds a vertical one.
2. Nami runs a frontier model and the Workers a mid-tier one: plan with the strong model, execute with the efficient one.
3. Every Worker is a persistent session I can inspect, not a short-lived black box.
4. I orchestrate through Nami and conduct through the Workers.

## Vertical context: why many sessions works

Each Copilot CLI session becomes a vertical expert in whatever it has been
working on. It holds the knowledge, the half-finished debugging and the
corrections I gave it earlier. So when something breaks in a resource
installation flow, I go straight to the session that has been living in that
flow. I don't re-explain anything. That part works well, and I would not go
back to one giant generic session.

## The cost nobody mentions

The cost is task switching. After jumping across five terminals, I sometimes
forget which one I owed a follow-up. One session is waiting for my answer,
another is waiting on a colleague, a third finished an hour ago and nobody
told me. The experts are fine. I am the bottleneck, and I am the one doing
the context switching.

## First, a dashboard

My first fix was simple: a small dashboard that tracks my Jira issues and my
active sessions in one place. That helped, but it only showed me the problem
more clearly. Then I noticed something. I already had a dashboard, and I
already had a personal assistant, Nami, who knows how I work. So why was I
the one managing all the other sessions?

We can already ask a Copilot CLI session to open another terminal and control
it through tmux. So I asked Nami to do exactly that, for every other session.
It worked on the first serious try, and I promoted Nami from personal
assistant to Chief of Staff.

<img src="{{ '/assets/nami-cos/horizontal-vertical.svg' | relative_url }}" alt="Diagram: you talk to Nami, the Chief of Staff, who holds the horizontal view across domains; each Worker holds a deep vertical context; a dashboard shows the Do-next list." style="width:100%; height:auto; margin: 0.5rem 0 1.5rem;">

## Horizontal context: what Nami holds

Nami does not go deep in any one domain. That is the Workers' job. Nami holds
the horizontal view: what I am doing across every domain, what is waiting on
whom, what is done, and what is next. Each Worker is deep and narrow, and Nami
is wide. Follow-up becomes easy and loose ends are much easier to spot,
because something is always keeping track of who owes whom a reply.

## How it works

The moving parts are small, and none of them is clever on its own.

1. **Butler** opens each Worker as a tmux session, which shows up as a terminal tab, named by workstream and a number. It sends the prompts and reads the replies through tmux.
2. **Dispatch** picks an idle Worker of the right domain, or spawns one if none is free, up to three per domain.
3. **Jobs** are the unit of work. Each one has a brief, a report and a state file. A Job also carries its Approvals: anything irreversible, like a publish, a merge or a post, stays a draft until I say yes in my own words.
4. **Declarations** are how a Worker talks back. It says "waiting on someone", "my move: here is the question", or "done".
5. **The dashboard** ranks a Do-next list from my tickets, pull requests, Jobs and those declarations.

Today there are sixteen tmux sessions live: one Chief of Staff and fifteen
Workers across about ten domains. Resource installation, an archiver tool,
the dashboard itself, code review, wiki, two flavors of our design
environment, AI tooling, my config, and GitHub admin.

## Plan with the strong model, execute with the efficient one

Nami, the Chief of Staff, runs on a frontier model, Opus 5.5. The Workers run on a mid-tier model, Sonnet 5.5. That
split is deliberate. I get to discuss, brainstorm and plan with a stronger
model, and then confidently let a medium-tier model execute the directions.

<img src="{{ '/assets/nami-cos/plan-execute.svg' | relative_url }}" alt="Diagram: the stronger model plans and writes a brief with explicit Approvals, and the mid-tier model executes the Job." style="width:100%; height:auto; margin: 0.5rem 0 1.5rem;">

The planning is where the strong model earns its cost. Choosing what to do,
splitting it into Jobs, and deciding which steps are safe and which need my
yes: that is judgment, and I want the best reasoner on it. Executing a
well-defined Job is a different kind of work, and a cheaper model does it
well.

What makes the cheaper executor reliable is the brief. Each Job gets a clear
brief from the planner, with the constraints spelled out and the Approvals
explicit: this step is a draft until the owner says yes, that one is fine to
run. A Worker that knows exactly where the line is does not need to be the
smartest model in the room. I only bring in the stronger model for a Worker
when a Job really needs deep reasoning, and most Jobs don't.

## Persistent sessions, not black boxes

In a typical orchestration setup, the sub-agents are short-lived and opaque.
They spin up, do something, return a result, and disappear. If the result is
wrong, you only get to guess why.

<img src="{{ '/assets/nami-cos/persistent-vs-opaque.svg' | relative_url }}" alt="Diagram: typical sub-agents are short-lived and return only a result; Workers are persistent sessions with history that can be attached to at any time." style="width:100%; height:auto; margin: 0.5rem 0 1.5rem;">

My Workers are different. Each one is a persistent session in its own tmux
terminal. It keeps its history, and I can open it any time and read exactly
what it did and why. That also means a Worker keeps getting better at its
domain across Jobs, instead of starting from zero each time.

## Orchestrate through Nami, conduct through the Workers

This is the part I like most, because I get both ways of working. For the
broad stuff I orchestrate through Nami: I dump everything I need to do into
one prompt window, and Nami dispatches each item to a Worker and gives me a
summary now and then, or I just look at the dashboard. When one Job needs real
attention, I conduct it myself: I step into that Worker's terminal, debug it,
and steer it directly. Nami is a layer on top, not a wall.

<img src="{{ '/assets/nami-cos/orchestrate-conduct.svg' | relative_url }}" alt="Diagram: you orchestrate through Nami for routine jobs and step into a Worker directly to conduct the one that needs attention." style="width:100%; height:auto; margin: 0.5rem 0 1.5rem;">

A normal evening looked like this. I dumped my list into the one prompt
window, and Nami dispatched:

- a dashboard bug fix and a new dashboard view
- a pull request review
- a script rework
- a GitHub team setup, coached interactively
- two release and capacity jobs
- the weekly report

Each one ran in its own Worker, and I kept a single conversation.

## A few things I learned

**Records beat memory.** Early on, the Chief of Staff carried a stale fact
over from a handover. It believed a task was still blocked when it had
already been done. A check against the session evidence caught it. The lesson
is plain: don't trust recall, trust records. So Nami now keeps durable Job and
journal records and checks them, instead of trusting its own memory.

**Reports come from records now.** My weekly report used to mean summarizing
every session transcript. Now it comes from the Job records. It is cheaper,
and more accurate, because the records say what actually happened.

## Where this goes

I am very satisfied with the results, even at this early stage. The next step
is to furbish the knowledge and playbooks. Once those are good enough, I want
Nami to pick up tickets and items from my todo list on its own, dispatch each to
a Worker, and attach a confidence score to the result, so I only have to look
closely at the ones that need me.

I have already shared the setup with my team. If you are running more than a handful of agent sessions, I
think the answer is not a better terminal layout. It is giving someone else
the job of remembering what is going on.
