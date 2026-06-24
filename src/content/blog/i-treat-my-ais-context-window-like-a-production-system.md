---
title: "I Treat My AI's Context Window Like a Production System"
subtitle: "Applying log rotation and cache eviction patterns to AI agent memory"
description: "Long-running systems need maintenance loops. Logs rotate, caches expire, databases compact. Your AI agent's context window is no different, but almost nobody treats it that way."
pubDate: 2026-03-25
heroImage: "../../assets/context-window-production.png"
---

Every production system accumulates state.

A server that never rotates its logs will eventually fill its disk. A cache that never expires entries will serve stale data with growing confidence. A database that never compacts will slow to a crawl under its own weight.

These are not edge cases. They are the predictable consequences of accumulation without maintenance. Every systems engineer knows this.

Now consider how most people run persistent AI agents.

Identity files, skill definitions, memory entries, tool outputs, conversation history. All loaded, all retained, nothing expired. The assumption is that more context means better performance.

For a single chat session, that assumption holds. The interaction is short. Context never accumulates long enough to degrade.

For an agent that persists across weeks, it eventually breaks.

---

## Context Degradation Looks Like an Intelligence Problem

When a long-running agent starts producing worse outputs, the instinct is to blame the model. The reasoning is weaker. The instructions aren't being followed. The agent seems to forget things it knew yesterday.

But the model hasn't changed. What changed is the environment it's reasoning in.

A context window stuffed with outdated memory entries, irrelevant skill definitions, and stale conversation fragments creates the same conditions as a server running on a full disk. The system isn't broken. It's suffocating.

The relevant information is still there. It's competing with everything that no longer matters.

This is not an intelligence problem. It is an attention allocation problem.

A half-empty context full of signal will outperform a full context full of noise.

---

## Four Ways Context Rots

The degradation is not uniform. It follows distinct patterns, each with a different root cause.

**Unbounded growth.** Memory entries accumulate session after session. Nothing expires. The store grows monotonically, and the index, the agent's inventory of what it knows, becomes a liability instead of an asset.

**Fragmentation.** Related facts written on different days remain separate entries. An infrastructure credential stored on Monday, a correction on Wednesday, and an update on Friday exist as three fragments when they should be one. The agent retrieves one and misses the others.

**Stale retention.** Debugging notes from two weeks ago still occupy context. A temporary workaround that was replaced still appears in the memory index. The agent treats outdated information with the same weight as current facts.

**Static overhead.** Identity files, skill definitions, and documentation are loaded verbatim every session. Written for humans, consumed by machines. Explanatory prose, redundant examples, and ASCII diagrams cost thousands of tokens the model will never act on.

These are not independent problems. They compound. Fragmented memory inflates the index. Stale entries increase context cost. Verbose documentation leaves less room for the entries that matter. Left unaddressed, they converge on the same outcome: an agent drowning in its own history.

Each of these failure modes has an established counterpart in systems engineering. And each has an established solution.

---

## The Maintenance Mapping

Production systems solved the accumulation problem decades ago. The patterns are well established. They map directly onto context management.

**Log rotation → Memory expiration.**

Server logs don't live forever. Entries older than a retention window are archived or deleted. The system keeps what's recent and relevant.

In the agent I run, memory entries tagged as daily are deactivated after 14 days. Not deleted, aged out. The durable facts remain. The transient observations expire. A nightly consolidation process handles this automatically, the same way logrotate runs on a cron schedule.

**Cache invalidation → Signal density tracking.**

A cache without TTLs accumulates stale entries that consume memory and return outdated results. Production systems track hit rates to identify what's worth caching.

The equivalent for agent memory is signal density:

```
signal_density = entries_accessed_24h / total_active_entries
```

When density is high, the store is lean. Most of what exists is being used. When it drops, the store is accumulating dead weight. In my system, a recent snapshot showed 3 entries accessed out of 23 active, a signal density of 0.13. The other 87% existed but contributed nothing.

Without this metric, memory bloat is invisible. With it, inefficiency becomes measurable and actionable.

**Database compaction → Nightly consolidation.**

Databases fragment over time. Related records scatter across pages. Periodic compaction reorganizes data for efficient access.

Agent memory fragments the same way. An insight written on Monday, a related correction on Wednesday, and a clarification on Friday exist as three separate entries when they should be one. The consolidation process I run nightly operates in four phases: re-topic misclassified entries using keyword rules, merge entries sharing the same topic and date, expire stale daily entries, and snapshot metrics to a log. It's idempotent. It runs unattended. It keeps the store from drifting.

**Resource budgeting → Selective loading and compression.**

A production application doesn't load every library into memory at startup. It loads what the current execution path requires.

My agent works the same way. Each skill declares trigger keywords. A request mentioning "pod" or "kubectl" loads the Kubernetes skill. A request about "invoice" loads the finance skill. If nothing triggers, only a one-line index entry is loaded instead of the full skill document. The overhead difference: a full skill load can cost 4,000 to 18,000 tokens. The index line costs 20.

For the files that are always loaded, identity and core instructions, a compression pipeline reduces token cost. The pipeline runs three strategies: a deterministic Python pass that strips prose scaffolding and keeps only structural content, an LLM-assisted rewrite at 40% of the original token count, and a second LLM pass that optimizes at the token level. If a compressed version fails validation against test queries, the system falls back to the deterministic pass. No silent degradation.

The result: documents that cost 12,000 tokens in their human-readable form cost 3,200 to 4,500 tokens when loaded into the agent.

The implementation details behind these patterns, including the memory store schema, the compression strategies, and the consolidation pipeline, are covered in [Memory Is Not Retrieval: Designing Long-Running AI Agents](/blog/memory-is-not-retrieval-designing-long-running-ai-agents).

---

## Why This Framing Matters

The current conversation around agent memory is dominated by retrieval. Better embeddings. Smarter vector search. More sophisticated ranking algorithms.

Those are valid problems when the knowledge base is large and heterogeneous. Thousands of documents where relevance cannot be anticipated.

But a persistent agent's working memory is different. It is small, structured, and written by the system itself. The challenge is not finding a needle in a haystack. The challenge is preventing the haystack from growing until it buries the needle.

This is an upstream problem. It cannot be solved at retrieval time.

Write-time filtering, where the agent itself decides what deserves persistence, is more effective than retrieval-time ranking for small, curated stores. The agent holds the task context. It knows what's durable and what's transient. The decision about relevance belongs where the context lives.

---

## The Result

None of this makes the model smarter.

What it does is protect the model's ability to reason.

A session that starts with 3,200 tokens of overhead instead of 12,000 has dramatically more space for actual work. Multi-step reasoning stays coherent longer. Instructions are followed more reliably. The agent doesn't "forget" things because the relevant context isn't buried under noise.

This is not a novel insight. It is the same insight that led to log rotation, cache TTLs, and database compaction, applied to a new substrate.

Long-running systems need maintenance loops. AI agent context is a long-running system.

The discipline already exists. We just haven't been applying it.

---

The context maintenance system described here is open-source: [context-engine](https://github.com/Jobenas/context-engine).
