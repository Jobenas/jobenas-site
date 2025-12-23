---
title: "Cutover Is Where Systems Stop Optimizing for You"
description: "Why system optimization works for the steady state but not for transient states."
pubDate: 2025-12-23
heroImage: "../../assets/migration.png"
---

### This wasn’t a database migration in the usual sense.

The system couldn’t stop.

Devices continued sending data. Background workers kept processing it. State kept evolving. There was no safe freeze point, no read-only window, and no “we’ll catch up later” option.

The task was to move a production IoT system to a new MySQL database while it remained live. Downtime had to be minimal—ideally zero. That single constraint shaped every decision that followed and ultimately exposed assumptions the system had never been forced to make explicit.

> The most important part was to keep downtime low.

We were keenly aware that devices would continue sending data throughout the process. The challenge was ensuring the handoff to the new database happened while all systems behaved as if nothing had changed.

Several weeks were allocated to planning. We mapped which tables were critical, which components would take longer to migrate, and how to sequence the cutover to minimize disruption—especially for device handling.

On paper, the plan looked straightforward: back up the database, migrate the critical and configuration tables quickly, then repoint services to the new database.

In practice, the cutover surfaced constraints that had never been exercised before—not because they were ignored, but because the system had never been forced to reveal them.

This wasn’t really a “database migration.”  
It was a live state transition under continuous load.

That realization drove the most stressful three days of operations I’ve had.

> **Systems are optimized for how they live, not how they change**

### When continuity becomes a correctness constraint

Some operations in the system depend on previous state when new data arrives. That meant the new database could not start empty, and the system could not tolerate a blind handoff.

This alone makes this class of migration fundamentally different from what most people imagine when they hear “production migration.”

Many such migrations quietly rely on at least one escape hatch:

- scheduled downtime  
- write freezes  
- read-only modes  
- paused consumers  
- dual-write periods with eventual reconciliation  

None of those were available here.

Once ingestion is continuous and downtime must be minimal, the problem changes shape. Time becomes part of correctness. Data arrival is no longer a variable you can suspend. “We’ll catch up later” stops being a universal answer.

At that point, *migration* becomes a misleading word.

What’s really happening is a **cutover**: a forced transition where steady-state assumptions stop being sufficient.

---

## Planning hits an epistemic limit

Several weeks of planning went into this change.

Schemas were reviewed. Data volume was scoped. A migration tool was designed and built to handle large tables, packet limits, partial history, and the timeline we had committed to. Secrets were prepared. Deployments were identified and staged.

And still, once execution began, everything else had to be dropped to deal with issues that only became visible during the cutover.

It’s tempting to conclude this happened because the system wasn’t probed deeply enough—that a more careful engineer would have surfaced everything in advance.

That belief doesn’t survive contact with how these systems actually behave.

Some constraints are not observable during dry runs or simulations. They only appear once the system is committed to a new phase.

Examples we encountered:

- network policies that appear correct until DNS resolution becomes critical under a new traffic path  
- indexes that are optimal for steady-state queries but pathological for bulk migration  
- data assumptions that hold indefinitely—until a time boundary is enforced  
- deployments that “apply” cleanly while touching none of the runtime paths that matter  

These aren’t oversights in the usual sense. They’re properties of systems that have only ever been exercised in one mode.

---

## Phase changes break steady-state designs

The index failures weren’t a skills problem. They were a phase problem.

Indexes that worked perfectly for day-to-day operation were actively hostile to large-scale data movement.

That feels like a rookie mistake. In hindsight, it’s obvious. But this isn’t about forgetting how indexes work. It’s about a system being optimized for one phase of its life and then forced—briefly and violently—into another.

OLTP-style indexes are designed for:

- selective queries  
- recent data  
- predictable write patterns  

Bulk migration stresses the opposite:

- wide scans  
- large inserts  
- delete-and-reinsert windows  
- index maintenance under sustained pressure  

There is no single index strategy that is ideal for both. Most systems don’t model this explicitly because they rarely need to.

**Until they do.**

What failed here wasn’t database knowledge. It was the assumption—shared by many systems—that the way data lives and the way data moves can be optimized the same way.

They can’t.

---

## When the system stops smoothing things over

There’s a moment during this kind of work where confidence drops sharply—not because the tasks are conceptually difficult, but because the system stops cooperating with your mental model.

Things that “should work” don’t. Things that worked yesterday fail for reasons that feel embarrassingly basic.

That sensation is easy to mislabel as incompetence.

A more accurate description is this: the system has stopped smoothing over its inconsistencies.

Distributed systems survive for long periods by accident. They rely on permissive networks, implicit defaults, historical data presence, and code paths that have never been forced to coexist under pressure.

Cutover removes that padding.

When connectivity, identity, data presence, and timing all become explicit at once, the illusion of simplicity collapses. The work feels junior because it is suddenly concrete.

---

## Operational correctness is not total correctness

Another distinction that only became obvious during the cutover was between **total data correctness** and **operational correctness**.

It was neither necessary nor realistic to migrate all historical data immediately. What mattered was:

- that the system could continue processing current data  
- that recent state required for operation was present  
- that failures were visible and attributable  

Older data still matters—for audits, analytics, and completeness—but it does not define whether the system is alive.

Treating these as separate concerns made the cutover possible. Pretending they were the same would have guaranteed failure.

This distinction is easy to state after the fact. It’s much harder to internalize when continuous ingestion meets a hard deadline—especially when you’re trying to preserve a usable end-user experience while guaranteeing near-zero downtime for device processing.

Some compromises are required. Those trade-offs were not observable during planning or dry runs.

> **Cutover is where unobservable constraints become observable**

---

## Cutover isn’t the point where plans fail

It’s the point where systems are finally forced to reveal constraints that were never visible while everything stayed nominal.

Planning reduces risk. It narrows the search space. It prevents avoidable mistakes.

What it cannot do is surface behaviors that only appear when the system is asked a question it has never had to answer before.

After the cutover, the system is in a better state than it was before:

- connectivity paths are explicit  
- configuration is verifiable  
- data scope is defined instead of implied  
- failure modes are louder and easier to attribute  

None of that came from elegance.

It came from forcing the system to commit.

If a migration feels calm all the way through, it probably wasn’t testing anything real.