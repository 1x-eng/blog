---
title: "Introducing Consult"
date: 2025-12-14T18:09:55+11:00
draft: false
---

# you can't surprise yourself. neither can your llm.

i've spent most of my career on backend systems. databases, apis, distributed stuff. the pattern i've noticed is that the worst production incidents don't come from bad code. they come from reasonable decisions that didn't account for something outside the decision-maker's domain.

a schema migration that's correct from a data modeling perspective but catastrophic from an operations perspective. an api design that's clean but creates n+1 query patterns the backend team didn't anticipate. a caching strategy that works until you hit a consistency edge case nobody thought about.

these aren't skill issues. they're perspective issues. one person, no matter how senior, optimizes for one dimension at a time. you can try to think about other angles, but you can't surprise yourself. you already know what you know.

the standard fix is peer review. get someone with different expertise to look at your design. the database person catches what the backend person missed. the infra person catches what the database person missed. this works, but it's expensive—you need access to those people, and their time.

i wanted to see if i could approximate that dynamic with llms.

## the problem with single-agent llm calls

the obvious first attempt (these days) afaict seem to be ask your claude's or gpt's or gemini's or whatever else is the latest 'hotshot' to review your architecture. afaict this mostly doesn't work for the same reason that asking yourself "what would the dba think?" doesn't work. the model is working from your framing. it optimizes for coherence with your prompt. as in, if you frame the question as a schema design question, you get schema design feedback. the locking semantics, the replication lag implications, the connection pool pressure - those live in a different frame.

you can try to prompt around this ("also consider operational concerns") but imo you get shallow coverage. the model isn't actually switching perspectives; it's appending considerations to the same perspective.

## what i built instead

consult runs `n 'agents'`, each with a domain-specific system prompt forcing them to stay in their lane and care deeply about that specific domain. (i.e. a database 'expert', backend 'expert', an infra 'expert', etc.). they analyze the problem independently, then each agent reviews each other agent's output. they vote on each other's work: APPROVE, CONCERNS, or OBJECT. if they don't hit the consensus threshold, they iterate with the feedback incorporated. finally, a synthesis 'agent' merges everything into one recommendation (without missing any trade-offs or objections, ensuring they are duly passed back to the user to consider)

the interesting part isn't any individual agent's output. it's what happens when they collide. say you ask about migrating a high-traffic table with dozens of foreign keys. the "database expert" (read: llm with a dba-flavored system prompt) proposes a dual-write pattern. the "backend expert" reviews it and flags that coordinating 47 fk updates under load needs application-level orchestration. the "infra expert" points out the wal replication lag from a multi-hour backfill. are they always right? nope - gotta take llms with a grain of salt. but they're each seeing different parts of the elephant, and that's the point.

single-agent responses smooth over these tensions. multiple agents with cross-review surface them.

## what this doesn't do

it doesn't replace knowing your domain. if you can't evaluate whether the agents' reasoning is sound, you're just adding latency to bad decisions.

it's not fast. 45-150 seconds, depending on the number of agents and iterations. that's n independent analyses plus n*(n-1) pairwise reviews plus synthesis. the latency is the feature - you're trading speed for coverage. and, that's the point of this tool.

it doesn't give you "the answer." it gives you a synthesis with the reasoning chains visible. you still have to read them and decide if they make sense.

## when it's worth the overhead

the decision framework i use: is the cost of getting this wrong higher than the cost of spending 2 minutes and 5-10x the tokens?

worth it:
- schema migrations you can't easily roll back
- auth/security decisions with compliance implications
- architectural choices that'll be expensive to change in 6 months
- anything where multiple domains intersect and you don't have easy access to experts in all of them

not worth it:
- syntax questions
- standard patterns that are well-documented
- anything you can undo in an afternoon

## an example

was designing active-passive failover for a payments service across two regions. standard setup - async replication to the standby, dns failover with health checks, the usual. i'd thought through the rpo/rto numbers, tested the failover runbook, felt good about it.

ran it through consult. the "infra expert" approved the failover mechanics. the "database expert" asked about in-flight transactions during the replication lag window - specifically, what happens to a payment that commits in the primary 400ms before the primary dies, but hasn't replicated yet? the "backend expert" pointed out that our idempotency keys were scoped to the primary's request-id generator. as in, after failover, a retry from a client could generate a "new" idempotency key on the standby and double-process a payment that actually succeeded on the now-dead primary.

that last one is subtle. it's not about the failover itself - it's about the interaction between our idempotency implementation and the failover semantics. i'd tested failover. i'd tested idempotency. i hadn't tested them together under a split-brain-adjacent scenario.

that's the kind of thing a single-perspective review misses. the infra person validates infra. the backend person validates backend. nobody's asking "what happens when these two correct implementations interact during a failure mode?"

## the cost

- 45-150 seconds latency (vs ~5 seconds for a single call)
- 5-10x token usage
- another cli tool to learn

what you get back is structured friction before you commit to something. whether that's worth it depends on how expensive the mistake would be.

## try it

consult is a cli tool (pro users get a `tui` interface). `pip install getconsult` [PyPi](https://pypi.org/project/getconsult/). and, consult is a `byok` model. as in, you bring your own api keys (anthropic, openai, or google), pay the providers directly. i don't see your questions, interactions, attachments or store your data. its all happening on your local. 

free tier gets you 2 experts, 3 queries/day. pro is $9/month for more experts, iterations, and features.

[getconsult.sysapp.dev](https://getconsult.sysapp.dev)

if you try it and it doesn't help, i'd like to know why: [issueTracker](https://github.com/1x-eng/consult-issues/issues)
