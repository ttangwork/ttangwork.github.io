---
layout: post
title:  "DevOps consulting: how clients waste the money they spend on me"
date:   2026-08-05
categories: consulting
---
I was brought into a project to help a client build a new platform, and I spent a good part of it raising ServiceNow tickets so that another team could click VMs and networks into existence by hand. Everything I built was code. Everything underneath it was somebody's afternoon in a portal. I could stand an environment up from a pipeline in minutes and then wait on a ticket for the subnet it was supposed to land in.

That is a fantastic use of an automation budget.

It's the clearest example I have of the thing that decides whether consulting money is well spent, and it has almost nothing to do with the consultant. A bucket holds water up to its shortest plank. It doesn't matter how good the other planks are, and it doesn't matter that I was hired to replace one of them. The clickops BAU was the shortest plank, so the platform ran at clickops speed, and the automation I delivered was capped by the slowest manual step it depended on. Fix the fundamentals first, or you're buying a tall plank for a short bucket.

Below are the other three ways I've watched engagements go to waste, drawn from client work over the years. None of them are technical.

## The handover is not the knowledge transfer

When you pay a consultant to do the work, the code and the tooling are the smaller half of what you're buying. The larger half is what your team can do after we leave, and that only accrues if we're embedded in the team while the work happens — in the same standups, on the same tickets, close enough that the feedback loop is a conversation rather than a document.

The failure mode is the big handover session a week before offboarding. I've been in those. They feel productive because a lot of information moves, and they achieve nothing, because information isn't the thing that was missing. The team hears it for the first time at the exact moment they're told to own it, and the honest reaction is shock, or denial, or a quiet decision to keep doing it the old way. The worst outcome isn't that the handover goes badly. It's that the team never touches the work at all, and you paid for something that gets rebuilt or abandoned in six months.

If you're the client, the question to ask isn't "will there be a handover". It's "who on my team has their hands on this today".

## You can't buy a culture

If your organisation doesn't know what DevOps is, buying the tools won't make you DevOps. You'll have the same problems, rebranded in nicer colours, with a dashboard on top. I can bring pipelines, IaC and a way of working, but I can't make two teams that have spent years protecting their boundaries start sharing responsibility for an outcome. That has to be driven from inside, by management, deliberately, and it means breaking silos that people are comfortable in.

It's easier said than done, and the honest part is that the middle is worse than anyone expects. There's a stretch where the old process is gone and the new one isn't trusted yet, everything takes longer, and it feels like the decision was wrong. That's the point at which most of it gets reversed. It's worth pushing through, but you should go in knowing that stretch is coming rather than discovering it and reading it as failure.

## Never trust a consultant

Including me.

I'm in this for your money. That's not a confession, it's the arrangement — but it means my incentive is to keep being useful to you, and your interest is in needing me less over time. Those aren't the same thing, and when they diverge it's your job to notice, because it won't be mine.

So treat the engagement as something you're accountable for rather than something you've outsourced. There's no quick fix and nothing changes overnight, whatever the proposal says. Fix your fundamentals so the good work isn't capped by the bad. Drive the cultural change from inside, because no external party can do it for you. Hire the right people for the right jobs. Make the hard calls that cut through the politics, which is the one that everyone agrees with and nobody does.

Do that consistently and you'll get real value out of people like me. Skip it, and you'll get a very well-engineered platform that your team doesn't use, delivered on time, on budget, and pointless.
