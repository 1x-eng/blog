---
title: "Point estimations and Sprints, huh"
date: 2023-07-27T17:47:42+10:00
draft: false
---

# Yet another post about software estimations - shouldn't it either be 1 point or NFI (No **** Idea)

There are things that engineers - especially software folk do well, we nail some really hard intangible problems on its head and boy, it feels good to build something ground up or cleanup a pile of mess. But, this group, almost holistically suck at one thing - that is estimation. 

Software engineers are notoriously poor estimators (well, its my blog so this is my view anyways). As an industry, we are not poor performers, although it may seem that way, because the prediction of performance is poor. There are many reasons why we set unreasonable expectations, and few will deny that estimating is a complex task. When performance doesn't meet the estimate, there are two possible causes: poor performance or poor estimates. In the software world, we have ample evidence that our estimates stink, but virtually no evidence that people in _general_ don't work hard enough or intelligently enough.

The impact goes far beyond cost overruns and missed deadlines. The typical approach to estimates ends up forcing bad behavior while privileging vanity metrics over delivering actual business value.

## Agile or Sprints to be precise correlates with Noise IMHO

In Agile environments, estimates are often based on story points and velocity. What can be so complex about creating a discrete piece of software anyways and in this age of _almost_ AGI, right? Well, if everything goes to "plan" and the GPTs were truly replacing human in the loop, sure - estimations should truly reflect velocity. But, its never been a perfect world and last time I checked, the promised utopia is still far fetched. 

This obsession with specifying and measuring the full process in advance wraps a plus or minus variance around a system that views engineers as machines pushing predictable work products through a pipeline at a steady stream. Yet to state what should be obvious, human beings are not machines. (Thank god! 😑) And maybe less obviously, the complexity of any _non-trivial_ software engineering task is almost impossible to accurately estimate in advance. This probably leads to - shouldn't we make non-trivial, .... more trivial? True, we should. And, for that, in true agile spirit, we create more tickets and assign points to these 'spike' tickets? Isn't that brilliant 😂

Sprints are where all of the issues with agile software dev really come from. Constant arbitrary, meaningless deadlines that serve no purpose but create rushes, tech debt, missed “commitments” that serve as a meaningless metric to use against teams while made commitments give no benefit. 

Sprints lead to meetings that attempt to plan better rather than actually getting work done. Nothing wrong with deadlines and drawing a line in the sand, they can infact serve a sense of motivation to get things done. However, with sprint ceremonies, often there is this 'calling' - measure everything you can which builds the idea that estimation is a skill that can easily be improved. That, is a 'cute wishful thinking' 😆, maybe a utopian fallacy? sure, you can learn to be less widely off, and this too is extremely contextual - it depends on skills & strengths of individuals that make a team. But, nooooo - we want to measure 'velocity'. What usually happens is that quality get shitty and corners get cut, and you get the product at the correct deadline, but it's not the same that you wanted to ship before development started (even if the features are nominally there).

## Whats with velocity? Isnt that a thing for cars, rockets and stuff?

One of the most common pre-requisites to sprints is estimating tasks (or so called "user stories" – what a name!). There are many different ways to do it, but the most common way I've seen is:

- have the whole team (💸) discuss the task at hand and agree on estimation (planning poker - and, people get paid to play this during work hours 😳)
- use a certain set of numbers to represent estimation values. This includes the works! Fibonacci series, t-shirt sizes and some creative minds also use tree viz to estimate tickets!

So, the outcome is some sorta key-value pairs, where every task has 'points' and these numbers are meant to represent ... something for someone or to everyone...? This number is also supposedly a 'guiding' principle for how much we can get done in x weeks/months - whatever is the sprint 'cycle' ♻️.

And, at the end of every sprint the objective is to look at the tasks that were delivered, they add up the number of the points of each task, and end up with a single number. This number represents how "fast" the team is going. It's a single number that represents all the team's work over a few months, and this number can be used to measure whether the team speeds up or slows down. Its often also used to compare performance of individual developers, to see who's driving the team's performance and who's lagging behind. Magic ✨ right?

... Well, absolute 💩. Velocity can provide value, but the way it's often used by Agile aficionados and inexperienced managers ruins the value of this metric. Why am I the antagonist here? Becuase:

### 🕞 My time !== Your time 

Points correlates with time. Time to finish the task. Developers are asked to predict how much time a task will take them. So 1 point might be a proxy for half a day, or a day. Then 2 points are 2 days, 5 points are a week. But, for some reason, the obvious here seem to be not so obvious - how much time I need to complete a task is different than how much time you need 😒. 

Some teams try to workaround this by assigning the tasks before estimating, and during estimation they ask "how much time it will take this specific developer to complete the task?" This approach is so bad it deserves its own article, but just to mention its biggest sin – it shifts from team-oriented to individual-oriented work, where everyone only cares about the tasks they have assigned to their name, instead of caring about all the tasks the team committed to.

### 🚀 More !== Better
Velocity = speed + direction, and the faster we move the better, no? Well yes, for cars, rockets and stuff yes. But, we are software programmers - i.e. human beings. Expecting developers to increase velocity over time contradicts the whole idea of estimating the complexity of tasks.

The base of this wrong assumption is that as the team gets better, they will deliver more value over time. And that's true, the team will build similar things faster, hence they will achieve more in the same span of time. But at the same time, as the team gets better and builds more sophisticated software, they will give lower estimations to similar tasks.

Velocity of the team over time should reach plateau and stay roughly the same, not grow (unless the team gets more people). 

### 🏭 Quantity !== Quality

> When a measure becomes a target, it ceases to be a good measure - Goodhart's law

Getting more done in a sprint, if that is 'incentivised', what that means is that the team will start inflating numbers, giving larger and larger values to the same type of tasks over time, just to show that the line on the velocity chart always goes up.

If the only measurement of success is velocity, then nothing else matters. Quality? Who cares, approve that Pull Request and let's go! Bug fixes? Only if a bug is assigned a number of points, otherwise it goes to the bottom of the backlog. The team works long hours and can't keep it going for long? Doesn't matter, the velocity must go up!


## 📍 1 point or NFI

IMO, for the majority of software development tasks, it's binary - it's either a trivial task that the individual or team (depending on their strengths, past experience or ability to get things done) is sure about, or it's complex enough that we're venturing into the unknown. I like to call it the "1 or NFI" approach - it's either 1 point, or No **** Idea.

The amount of unpredictability and volatility inherent to our job makes an accurate estimate almost a contradiction in terms. Is it a bug? Oh, it could be a walk in the park, or a deep rabbit hole of despair. New feature? Could be a few lines of code, or a total cluster**** of interdependent systems. Refactoring? Could take a day or two, or you might open a Pandora's box that makes you wish for a career change.

So, am I advocating for no planning, but reckless and procrastination friendly workplace? Maybe not. Understandably, the need for ballpark figure to help the business plan its strategy, set expectations and all that jazz is ... you know - understandable. But, my point here is - why are we forcing a sham precision on an inherently fuzzy problem in the name of adopting agile? If we know it's a trivial task, sure - give it a 1 point. But if it's a non-trivial task, it's NFI. It's like trying to measure the surface of the ocean with a ruler - it's just the wrong tool for the job.

Stuck with an NFI task? Find the 'doer(s)' in your workplace and attempt to break it down (if possible). In other words, try and turn the NFI task into something that we can confidently say is a 1 point. It's moot to bring the whole team to 'estimate' these tasks where John doe & John snow can talk about their opinions non-stop while the others browse through their instagram / tik-tok / threads / twitter / x whatever else is the thing they find amusing 🔫

When we throw out the illusion of precision, we can actually focus on delivering value. With the 1 or NFI approach, we get rid of useless discussions about whether it should be a 2 or a 3. Instead, we can focus on problem-solving, on understanding what we don't know and figuring out how to know it. We can give meaningful updates to our stakeholders: "we're still figuring out how to solve this" is a lot more meaningful than "it's 80% done" for the third week in a row.

In the end, the 1 or NFI approach is not just about estimates. It's about facing the reality of software development: it's complex, it's unpredictable, and it's hard. And, please stop playing poker with 'sprints' 🎲

