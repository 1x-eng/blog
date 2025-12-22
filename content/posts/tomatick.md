---
title: "Tomatick"
date: 2024-10-30T14:59:47+11:00
draft: false
---

# tomatick

meh... what's with the name? it's "tomato" + "tick" (as in timer). dad joke? probably.

## the problem

every productivity tool promises to make you more efficient. most fail. at least, that's been my experience. why? because they treat our brains like input-output machines: set a timer, get focused work. reality is messier.

the average knowledge worker:
- loses ~23 minutes to each context switch (estimates vary, but it's not zero)
- makes thousands of decisions per day, each one draining the tank a little more
- has optimal focus periods that rarely align with whatever arbitrary timer you set
- can get stuck in perfectionism-induced paralysis (this one's personal - i think "being a perfectionist" is treated as a badge of honor in our industry. it's not. it's a trap. maybe a disorder. i'll save that rant for another post.)

## why i built this

i've tried the usual suspects - forest, focus@will, be focused, and a dozen others. they're well-designed, but they all share the same fundamental limitation: they assume productivity is a timer problem.

the irony? a productivity tool should work for you, not the other way around. you shouldn't need a separate system just to figure out if your productivity system is working.

so instead of building yet another timer, i asked a different question: how does the brain actually work during complex cognitive tasks, and how can we design around that?

## what's inside

three core ideas:

**1. pattern recognition**
- monitors cognitive load over time
- adapts to how you actually work (not how some productivity guru says you should)
- suggests task sequencing based on your patterns
- tries to prevent decision fatigue before it hits

**2. momentum building**
- sequences tasks for flow state entry (or at least attempts to)
- minimizes context-switching
- maintains cognitive momentum without burning you out

**3. burnout prevention**
- tracks energy expenditure patterns
- forecasts cognitive fatigue
- suggests breaks based on actual load, not arbitrary intervals

## memory integration (optional)

traditional productivity tools treat each session as isolated. but our brains don't work that way - we build on previous experiences and develop intuition over time.

tomatick works fine standalone. but if you want the memory layer, it integrates with [mem.ai](https://mem.ai). not affiliated, just a fan. their product creates a web out of your thoughts, and thanks to llms, you can interact with that knowledge graph naturally.

fair warning: mem.ai is rough around the edges. they call it production-grade, but... let's say it's a work in progress. still, i haven't found anything else that approaches personal knowledge management quite this way. (and no, obsidian doesn't count unless you have infinite time to configure it.)

if you do use it, the integration means:
- your tasks connect to your notes, research, and thoughts
- temporal search lets you find patterns across time ("what was i working on last quarter?")
- productivity data becomes part of your knowledge base, not a separate silo

## does it actually work?

early testing (just me, ~3 months, 8-10 hour workdays across professional work, side projects, and personal stuff) shows:
- fewer context-switching spirals
- less perfectionism-induced stalling
- better energy management throughout the day
- and honestly? it's something when a tool tells you "can you please call it a day? you've been killing yourself. it's not the end of the world if you don't finish that one thing today."

## what's next

the current version is just a start. on the list:
- health data integration (garmin first, maybe fitbit later) to correlate productivity with sleep, stress, etc.
- better temporal pattern analysis ("when i worked on something similar last time, what worked?")
- deeper mem.ai integration

## try it

if you've read this far, thanks. tomatick is FOSS:
- [github](https://github.com/1x-eng/tomatick)

ps:
- it's a cli tool
- i built this for myself while working on something else. work in progress.
- if you want an app or website, sorry for wasting your time.
