---
title: "SKIP LOCKED vs CAS in PostgreSQL"
subtitle: "Two ways to let many workers claim rows without stepping on each other: the overview"
date: 2026-10-06T00:00:00+01:00
lastmod: 2026-10-06T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "FOR UPDATE SKIP LOCKED and compare-and-swap (optimistic) updates both stop two workers claiming the same row, but behave very differently under contention. An overview with a measured result on PostgreSQL 16: 8000 of 8000 claims vs 2745 of 8000. Part 1 of 3."
license: ""
images: []

tags: ["postgresql", "concurrency", "system-design", "queues", "sql"]
categories: ["system-design"]

featuredImage: "/images/posts/postgres-skip-locked-vs-cas.png"
featuredImagePreview: "/images/posts/postgres-skip-locked-vs-cas.png"

hiddenFromHomePage: false
hiddenFromSearch: false
---

Eight workers poll the same `jobs` table. Each job must be processed by exactly one of them. Two PostgreSQL tools solve this, `FOR UPDATE SKIP LOCKED` and compare-and-swap (CAS), and on the same workload one claimed every job it tried for while the other lost most of its attempts.

<!--more-->

This is the overview. Each strategy gets its own deep dive: [SKIP LOCKED explained](/posts/postgres-skip-locked-explained/) and [CAS explained](/posts/postgres-cas-explained/).

## The Problem

```sql
CREATE TABLE jobs (
  id      bigserial PRIMARY KEY,
  status  text NOT NULL DEFAULT 'pending',
  owner   text,
  version int  NOT NULL DEFAULT 0
);

CREATE INDEX jobs_pending ON jobs (id) WHERE status = 'pending';
```

The obvious claim is two steps, a `SELECT` then an `UPDATE`:

```sql
SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1;
UPDATE jobs SET status = 'running', owner = 'w1' WHERE id = 42;
```

Between them, another worker can run the same `SELECT`, get id 42, and run the same `UPDATE`. Both believe they own job 42. Both strategies close that gap, in opposite ways.

## The Two Ideas

| | **`FOR UPDATE SKIP LOCKED`** | **CAS** |
|---|---|---|
| **Style** | Pessimistic: lock the row, then change it | Optimistic: change it only if it's still what I saw |
| **Loser's experience** | Never sees the row, takes the next one | Waits briefly, re-checks, updates 0 rows |
| **Needs a retry loop** | No | Yes |
| **Row picked** | Any unlocked one | Exactly the one you named |

## The Measured Result

I ran three claim strategies with `pgbench` on PostgreSQL 16: 20,000 pending jobs, 8 clients, 1,000 claim attempts each.

| Strategy | Claims succeeded (of 8,000) | Wasted attempts |
|---|---|---|
| `FOR UPDATE SKIP LOCKED` | **8,000** | 0 |
| CAS on the head row (single statement) | **2,745** | 5,255 (66%) |
| CAS on a random one of the first ~200 pending rows | **7,799** | 201 (2.5%) |

One run on a laptop, a local socket, `fsync=off`, no work between claims. It shows the *shape* (naive CAS degrades with contention, SKIP LOCKED doesn't), not your production throughput. Raw speed was similar for all three (roughly 9,000 to 12,000 statements per second), which is why the wasted attempts are the number that matters.

## Which One

| The situation | Reach for | Because |
|---|---|---|
| Many workers draining a job queue | `SKIP LOCKED` | No contention on the head row, no retry loop |
| A user edits a record others may edit too | CAS on `version` | You must act on *that* row and detect that it changed |
| A state transition that must happen once (`pending` → `paid`) | CAS on `status` | The `WHERE` clause is the guard |
| Queue throughput matters *and* you need safe completion | **Both** | `SKIP LOCKED` to claim, CAS on `owner` and `version` to finish |

They aren't rivals. SKIP LOCKED decides *who gets which row*, and CAS makes sure the *finish* is only accepted from the worker that still owns it.

## Next

- [SKIP LOCKED explained](/posts/postgres-skip-locked-explained/): how it works, what it costs, and its pitfalls.
- [CAS explained](/posts/postgres-cas-explained/): why it blocks, why it wastes work on a queue, and where it is the right tool.
