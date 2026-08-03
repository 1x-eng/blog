---
title: "On Database Internals"
date: 2026-08-03T19:21:03+10:00
draft: false
---

i wrote [database is sacred](https://1x-eng.github.io/blog/posts/database-is-sacred/) back in 2023. the gist was to stop using your rdbms as a dumping ground - normalise rather than reaching for a json column, do the aggregation in the database instead of pulling everything into the application, wrap multi-step operations in a transaction. i'd still argue all of it. what's obvious to me now is that i had no real idea what happens between calling `commit` and the data being on disk, so the whole post was me generalising from things i'd watched break. the conclusions were fine. i couldn't have properly defended any of them if someone had pushed.

most of the time it didn't come up. it came up when someone senior disagreed and the only thing i had was that i'd watched it go wrong before, which isn't much to stand on.

alex petrov's *database internals* is the book that fixed that for me. it didn't change my mind about much and i came out of it holding roughly the positions i went in with. what changed is that i can show my working now.

## why a write-ahead log exists

if you'd asked me before, i'd have said "durability" and left it there. that's true, but it doesn't explain why the thing exists.

the wal exists so the engine can decouple when your transaction commits from when your data pages actually hit disk.

it writes some of your data **early**, before you've committed, because it wants the memory back and isn't going to sit there waiting for your transaction to finish. if the power goes at that moment, your data files contain changes nobody ever committed, so the engine has to be holding enough information to undo them.

it writes the rest **late**, well after you've committed, because making commit wait on a scattered handful of random pages would be miserable. if the power goes at that moment, your data files are missing changes that were committed, so it has to be holding enough information to redo them.

commit, then, never waits on your data pages. it waits for one fsync on the log, and that's the whole budget.

that's what group commit is. a handful of transactions committing at the same moment are queued behind that same fsync, so they share it instead of each paying for their own. and if one fsync is the entire cost of committing, then the disk the log sits on *is* your commit latency. that's the reason to keep it off the data files, so random page writes aren't fighting sequential log writes for the same io.

it also tells you what `synchronous_commit = off` actually costs. with it off, a crash can lose transactions that already returned success to the client, bounded by the flush interval. it won't corrupt anything - postgres's docs put the resulting state as "just the same as if those transactions had been aborted cleanly." which makes it a reasonable trade for event ingestion or metrics, and a bad one anywhere a lost commit means a lost payment.

## the same statement doesn't cost the same twice

sql hands you rows. the engine underneath is working in pages, 8kb in postgres unless you've changed it, and almost every cost you care about is a page cost rather than a row cost.

take the smallest write you can think of, setting one boolean on one row. what that costs depends on state that doesn't appear anywhere in the statement you wrote. if no indexed column changed and there's free space on the page, you get a heap-only tuple update, which puts the new version on the same page and writes no index entries at all. (summarising indexes are excluded from that condition, which in core postgres means brin and nothing else.) if either of those conditions fails, every index on the table has to be updated. and if this happens to be the first modification of that page since the last checkpoint, `full_page_writes` is on by default and the entire 8kb page gets written into the wal, because a page that was half-written when the machine died can't be reconstructed from a row-level record.

so the same statement costs different amounts depending on fillfactor, on which columns happen to be indexed, and on how long it's been since a checkpoint. it also costs different amounts depending on which engine you're on. postgres appends a new tuple version, innodb updates the row in place and pushes the previous version into undo, oracle does something different again. different write amplification, different behaviour under a long-running reader, and a cost model built on one of them will mislead you on the next.

i used to ask whether a query was slow. now i ask how many pages it touches and how many of those get written.

"the database is locking up" is what gets said during an incident, and a decent share of the time the contention isn't on locks at all. **latches and locks are different things.** locks are logical: they protect rows and key ranges, they're held for the length of a transaction, they live in the lock manager and they show up in deadlock detection. latches are physical: they protect a page's in-memory representation while an operation is using it, and there's no deadlock detection because they're acquired in a fixed order, so deadlock never comes up in the first place. latch contention, buffer pool contention and wal writer contention don't appear in `pg_locks` and don't improve if you go hunting for lock contention.

## sequential keys cut both ways

i thought i understood why people like time-ordered ids. turns out i only knew half of it.

the half i knew is that b-trees have a fast path for keys that only ever go up. sequential inserts land at the rightmost leaf, split cleanly, keep the hot part of the tree in cache and produce dense pages. random keys, uuidv4 being the usual culprit, land uniformly across the whole tree, so you get splits everywhere, no locality on the insert path, and an index that grows faster than the table it's indexing.

the half i'd missed is that those same sequential inserts are all landing on the same leaf, so they're all contending for the same latch and the chain of latches above it. it's the same property doing both jobs. they're cheap because they all go to one place, and they contend because they all go to one place. one layer up from the engine, in anything partitioned, a key that always increases is the standard way to end up with a hot shard - cassandra partition keys, dynamodb, hbase region splits, they all fail this way and for the same reason each time.

so what you want is temporal locality *within* a partition and spread *across* partitions. uuidv7 or ulid gets you the first. a partition key that isn't the timestamp gets you the second. knowing only the first half points you in exactly the wrong direction once contention is the limiting factor.

## compaction, which nobody owns

compaction strategy gets set once, by default, and then never looked at again, because it files under tuning and tuning belongs to nobody. it's where an lsm's write and space amplification actually get determined.

**size-tiered** merges tables of roughly similar size. write amplification stays low. space amplification is high but temporary, because while the merge is running you need room for the output alongside the inputs that produced it. that's where the advice about keeping serious free headroom on cassandra nodes comes from. the real bound is the size of whatever tables are being merged rather than a fixed fraction of the disk, though the fixed fraction is a much safer thing to hand to whoever's on call.

**leveled** keeps non-overlapping ranges within each level, which bounds space overhead to roughly 10%, because about 90% of your data ends up in the last level. you pay for it in write amplification, since every byte gets rewritten on the way down. i'd be careful with the number here. per level it's the fan-out in the *worst* case, and rocksdb's own documentation says it "tends to be less than the fanout in practice," so fan-out times levels is a ceiling rather than something to plan against. the only figure rocksdb will actually commit to is "often larger than 10."

"lsms are fast for writes" is true amortised, and only for as long as compaction keeps up. compaction is itself a large background write amplifier competing with your foreground traffic for the same io budget, and if sustained write load outruns it, rocksdb will stall writers outright rather than let read amplification run away. an lsm converts a steady write cost into a deferred, bursty one, and whether that's a good deal depends on whether your workload gives compaction enough quiet time to catch back up.

"b-trees are read-optimised" is true for lookups against a warm buffer pool. once your random-read working set is bigger than memory, both structures are doing io per lookup and the comparison turns into a question about layout and caching rather than about the shape of the tree.

there's a framing for all of this called the RUM conjecture - reads, updates, memory overhead, pick two. it's a *conjecture* rather than a proven bound (and it comes with no exchange rate), so, afaiu, all it really says is that every optimisation moves cost somewhere else. i use it as a smell test now. when something claims to be good at all three at once, afaiu, it isn't whether that's possible anymore, but it's whether my workload is the one paying it!

## isolation levels don't mean the same thing twice

two transactions read the same set of rows. each one checks that some rule still holds, then writes a different row on the strength of what it read. neither one touched what the other wrote, so nothing conflicts and both commit. the rule they were jointly enforcing is now broken.

that's write skew. snapshot isolation never had a reason to stop either transaction, and snapshot isolation is what postgres gives you when you ask for `REPEATABLE READ`. the shape to watch for is any constraint of the form "at least one of these has to stay true", checked by reading the set and then writing your own row. if you work anywhere near money, imho that's a shape worth going and looking for.

the underlying problem is that the level names are close to meaningless across engines. the sql standard defines them by which anomalies they have to prevent rather than by what the engine does, so "we run at repeatable read" tells you nothing portable.

postgres `REPEATABLE READ` is snapshot isolation, with a transaction-scoped snapshot, first-updater-wins, and serialization failures on write conflict. innodb `REPEATABLE READ` is a hybrid, where a plain `SELECT` reads a consistent snapshot but locking reads and writes see the latest committed row and take next-key locks on the gaps. mysql's own documentation suggests not mixing locking and non-locking statements inside one such transaction, on the grounds that if you're doing that you probably wanted serializable. the words are identical and the guarantees aren't, so code that's correct on one engine isn't necessarily correct on the other!

postgres `SERIALIZABLE` genuinely is serializable. it's SSI underneath, tracking read-write dependencies and aborting transactions that would complete a dangerous cycle. so it's a different mechanism from `REPEATABLE READ` rather than a tighter setting of the same one, even though the naming implies a ladder. it also requires your application to handle retries, which makes moving to it a code change rather than a configuration change.

# once there's a network involved

## every timeout is a guess

you cannot tell a slow node from a dead one. not with better monitoring, not with tighter timeouts. no amount of tooling closes that gap, because it's a proof: in an asynchronous system where even one process can crash, no deterministic algorithm guarantees consensus. every practical system gets around it by quietly assuming some bound on how late a message can be, and the failure detector is where that assumption lives.

a detector gets judged on two things. completeness, meaning every crashed process is eventually suspected, which is cheap because you can get it by suspecting everybody. and accuracy, meaning correct processes aren't wrongly suspected, which is the expensive half and unachievable in its strong form, so what people actually use are the eventually-accurate ones.

phi-accrual, which cassandra and akka both use, afaict, changes the question. coz, instead of asking whether a node is dead, it asks how surprising the gap since the last heartbeat is, given what that link has actually been doing. you turn the answer back into a boolean anyway, since cassandra has `phi_convict_threshold`, but the threshold is now in units that adapt. a fixed five-second timeout is simultaneously too twitchy on a congested link and too slow on a healthy one, and no constant fixes both.

it's still wrong sometimes. deciding a node is dead is a probabilistic call and everything you build on top of that decision inherits the probability, which is why the consensus protocols don't depend on the detector being right. they depend on a number that only ever goes up - paxos ballot numbers, raft terms - so even when the detector gets it wrong, a demoted leader can't make progress, because everyone else rejects work carrying an older number.

i wrote about [consult](https://1x-eng.github.io/blog/posts/introducing-consult/) turning up an idempotency bug in an active-passive payments failover, where a client retry after failover could mint a new idempotency key on the standby and double-process a payment that had already committed on the primary that just died. i'd tested failover. i'd tested idempotency. i hadn't tested them together. what i didn't have at the time was the language for why that's structural rather than bad luck. the failover happened because a detector made a probabilistic call, and the application then treated that call as established fact. improving the detector doesn't help. the fix is to derive the idempotency key's scope from something that survives the transition, and to make sure any write path can be rejected by epoch once a node has been demoted.

## where the coordination lives

"distributed transactions" isn't really a feature. every system coordinates somewhere, and the interesting question is where. afaiu, there are four answers and every distributed database picks one of them.

**2pc** puts the decision in a coordinator. if the coordinator dies after the cohorts have voted to prepare but before it broadcasts the outcome, those cohorts are stuck holding locks and can't work it out among themselves, because none of them knows whether some other cohort voted no. **3pc** adds a pre-commit round which removes that blocking under crash-stop failures, but it buys that by becoming incorrect under a network partition, where the two sides can independently reach different decisions. so it's a different trade rather than an improvement, and you almost never see it deployed.

**calvin** moves coordination in front of execution. a sequencing layer fixes the order of transactions first, and then every replica works through that same order and ends up in the same state, so there's nothing left to agree about afterwards. what you give up is flexibility: you have to know the read and write set before you start, so if you don't know what you're going to read until you've started reading, the model doesn't work. faunadb was built on this.

**spanner** buys coordination with clocks. truetime hands you an uncertainty interval rather than a timestamp, and commit-wait deliberately waits that interval out so that timestamp order is guaranteed to match real-time order. you get external consistency and you pay for it in latency proportional to that uncertainty bound, which is why google spends what it does on gps receivers and atomic clocks. better clocks don't just make the timestamps tidier, they directly cut commit latency.

**percolator** puts snapshot isolation on top of a plain key-value store, using client-driven 2pc with a primary lock row and a central timestamp oracle. tidb comes from this lineage.

so when a database says it does distributed transactions, the thing to work out is which of those four it means. the failure modes come with the answer.

## consensus does not give you mutual exclusion

consensus gives you agreement about the contents of a log. it gives you nothing whatsoever about the world outside that log. a process can be holding a perfectly valid lease, get stopped for a while by a gc pause or vm steal or a network blip, have the lease expire and get granted to someone else, and then wake up and carry on acting as though it still holds it. nothing about the protocol failed here. the process assumed its own view of the world was still current, and it wasn't. the only real defence is that whatever resource you're protecting refuses to accept work from a stale actor, which means checking an epoch, or a fencing token if you've met the idea under that name. a lock service can tell you who holds the lease. it can't stop something that's already forgotten it lost one.

## harvest and yield, instead of CAP

cap is a narrow result about a specific model. it says nothing about how a system should behave when there isn't a partition, which is almost all of the time.

harvest is the fraction of the data reflected in your answer. yield is the probability of completing a request. a search index that returns results from 8 of its 10 shards has degraded harvest and full yield. a system that refuses writes during a partition has full harvest and degraded yield.

harvest and yield are properties of an operation, not of a system. the same service can be perfectly happy to trade harvest on a feed and completely unwilling to give up a single point of it on a balance query. so the design review question stops being "is this AP or CP" and becomes "what harvest and yield do we want out of this endpoint", which is something you can actually write down.

# where it's thin

the thing that genuinely annoys me is how little quantification there is. this is a book about mechanisms whose entire significance is cost, and it will tell you that leveled compaction has higher write amplification without telling you whether that means three times or fifty times. you end up in the papers and the rocksdb documentation anyway, which is fine, but i'd rather it said so instead of leaving you with a direction and no magnitude.

the cost model throughout assumes locally-attached block storage. that isn't a complaint about the book's age, since aurora's paper predates it. it's that disaggregation isn't the frame anywhere in it - compute and storage separation, pushing redo application down into the storage layer so the compute node never writes a data page at all, object storage as the primary tier. a lot of the last decade went in that direction and none of the reasoning here is calibrated for it. vector indexes aren't in it either, which is entirely fair for 2019 and still a gap if you're reading it now.

some stretches also read as annotated bibliography rather than explanation, particularly the b-tree variants material and the tour through the paxos family 🤷‍♂️

# what i'd take back from my "database is sacred" post

the central claim holds up, and i can now give a reason for it rather than an observation. an aggregate computed at the source moves a result back to the application; the same aggregate computed in the application moves every page you're about to throw away.

the postgres `RULE` i suggested for audit logging, though, i'd withdraw. rules aren't triggers. a rule rewrites your query rather than firing per row, and the documentation warns about the consequence that i'd missed before.. i.e. "rules tend to have surprising results when the original query contains volatile functions: volatile functions may get executed more times than expected." the same page says that things done with rules on insert, update and delete are in many cases better done with triggers. to be fair to the other side, the docs also point out that a rule can beat a per-row trigger when a statement affects a lot of rows - but the example in that post inserts one row per statement, so that advantage was never available to it in the first place.

the broader thing i'd revise is that "push logic into the database" and "push logic somewhere with worse version control, worse testing and worse observability" can describe the same change, and at the time, the post didn't distinguish between them. views and normalisation are straightforwardly right. stored procedures and rules are a genuine trade-off, and i wrote about them as though they were free.

# so, worth reading?

the storage engine half, yes, and slowly. recovery and log-structured storage are the two worth reading twice, and there's almost nothing else published between "here's what an index is" and reading postgres source directly. the distributed systems half only if you haven't read ddia, and even then read ddia first, because it's a decent survey of ground kleppmann already covers more patiently. the further reading lists at the end of each chapter are worth more than several of the chapters themselves, imho.

it hasn't made me faster at tuning anything and there's no checklist of production optimisations in it, so if that's what you're after this isn't the book. it replaced a pile of rules of thumb with mechanisms i can reason from.
