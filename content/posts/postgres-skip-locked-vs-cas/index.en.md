---
title: "SKIP LOCKED vs CAS in PostgreSQL"
subtitle: "Two ways to let many workers claim rows without stepping on each other, and what each one costs"
date: 2026-10-06T00:00:00+01:00
lastmod: 2026-10-06T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "FOR UPDATE SKIP LOCKED and compare-and-swap (optimistic) updates both stop two workers claiming the same row, but they behave very differently under contention. A measured comparison on PostgreSQL 16: 8000 of 8000 claims vs 2745 of 8000, plus when to use which."
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

This post compares them on one problem, a worker claiming a pending job, with a runnable experiment so you can check the numbers on your own machine.

## Bottom Line

| | **`FOR UPDATE SKIP LOCKED`** | **CAS (`UPDATE … WHERE status = 'pending'`)** |
|---|---|---|
| **Style** | Pessimistic: lock the row, then change it | Optimistic: change it only if it's still what I saw |
| **Loser's experience** | Never sees the row, moves on to the next one | Blocks briefly, re-checks, then updates 0 rows |
| **Wasted attempts under contention** | None: every query claims a row if one is free | Many, if workers all target the same row |
| **Needs a retry loop** | No | Yes |
| **Row picked** | Whatever isn't locked, so strict ordering is not guaranteed | Exactly the one you named |
| **Best when** | Many workers drain a queue | One specific row, such as an edit, a state transition, or a version check |

If you only remember one line: **SKIP LOCKED is for "give me any available row". CAS is for "change this particular row, but only if nobody else already has".**

## The Problem

```sql
CREATE TABLE jobs (
  id      bigserial PRIMARY KEY,
  status  text NOT NULL DEFAULT 'pending',
  owner   text,
  version int  NOT NULL DEFAULT 0
);

-- keeps "find the next pending job" cheap as the table fills with finished work
CREATE INDEX jobs_pending ON jobs (id) WHERE status = 'pending';
```

The obvious approach is two steps:

```sql
SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1;
-- ...application code...
UPDATE jobs SET status = 'running', owner = 'w1' WHERE id = 42;
```

Between the `SELECT` and the `UPDATE`, another worker can run the same `SELECT`, get id 42, and run the same `UPDATE`. Both believe they own job 42. Everything below is a way to close that gap.

## Approach 1: FOR UPDATE SKIP LOCKED

**The idea:** select a row *and lock it* in the same statement. Any other worker whose query hits a locked row doesn't wait for it. It skips it and takes the next one.

```sql
WITH next AS (
  SELECT id FROM jobs
  WHERE status = 'pending'
  ORDER BY id
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UPDATE jobs j
SET status = 'running', owner = 'w1', version = version + 1
FROM next
WHERE j.id = next.id
RETURNING j.*;
```

One statement, one round trip. If there's any unlocked pending row, you get it. If the result is empty, the queue really is empty (or fully claimed).

**Strengths:** no wasted work and no retry loop. Throughput scales with the number of workers because they never fight over the same row.

**Weaknesses:**

- **Ordering is not guaranteed.** If worker A holds job 1 and worker B skips it to take job 2, job 2 can start before job 1. Fine for independent jobs, wrong if order matters.
- **It sees an inconsistent view on purpose.** The PostgreSQL docs say this is fine for queue-like tables and not suitable for general-purpose work. Don't use it to answer "what is the oldest pending row?".
- **Don't hold the lock while doing the work.** If you `BEGIN`, claim, process for 30 seconds, then `COMMIT`, you've pinned a transaction open the whole time. The pattern above avoids that by claiming with a status change and committing straight away.

## Approach 2: CAS

**The idea:** don't lock anything up front. Try the change with a condition that is only true if nobody beat you to it, then check how many rows you changed.

```sql
UPDATE jobs
SET status = 'running', owner = 'w1', version = version + 1
WHERE id = 42 AND status = 'pending';
```

If it reports `UPDATE 1`, you won. If it reports `UPDATE 0`, someone else got there first, and you retry.

Under the default `READ COMMITTED` isolation, here is what actually happens when two workers run this at once. The second `UPDATE` blocks on the row lock held by the first. When the first commits, the second **re-evaluates its `WHERE` clause against the new row**, sees `status = 'running'`, and updates nothing. So CAS in Postgres is not lock-free. It's a short wait followed by a re-check, and the `WHERE` condition is the "compare" in compare-and-swap.

The same shape works for versioned edits, where the condition is `WHERE id = 42 AND version = 7` and you prevent lost updates on user-edited records.

**The catch for queues is picking the row.** The tempting single-statement version:

```sql
UPDATE jobs SET status = 'running', owner = 'w1', version = version + 1
WHERE id = (SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1)
  AND status = 'pending';
```

Every worker's subquery picks the *same* head row. One wins. The rest wait, re-check, fail, and report `UPDATE 0`, even though thousands of other pending rows exist.

## The Experiment

I ran three claim strategies with `pgbench` on PostgreSQL 16: 20,000 pending jobs, 8 clients, 1,000 claim attempts each (8,000 attempts total). The only thing measured is how many attempts ended up owning a job.

| Strategy | Claims succeeded (of 8,000) | Wasted attempts |
|---|---|---|
| `FOR UPDATE SKIP LOCKED` | **8,000** | 0 |
| CAS on the head row (single statement) | **2,745** | 5,255 (66%) |
| CAS on a random one of the first ~200 pending rows | **7,799** | 201 (2.5%) |

The third row is the fix for naive CAS: spread the workers out, so they rarely target the same row.

```sql
\set off random(0,200)
SELECT id AS cid FROM jobs WHERE status = 'pending' ORDER BY id OFFSET :off LIMIT 1 \gset
UPDATE jobs SET status = 'running', owner = 'w' || :client_id, version = version + 1
WHERE id = :cid AND status = 'pending';
```

Caveats on these numbers: one run on a laptop, a local socket, `fsync=off`, no work done between claims, and 8 workers. They're meant to show the *shape* (CAS degrades with contention, SKIP LOCKED doesn't), not to predict your throughput. Raw speed was in the same ballpark for all three (roughly 9,000 to 12,000 statements per second), which is why the 66% waste is the number that matters: the fast CAS run was fast largely because it was doing nothing.

## Try It Yourself

```bash
# throwaway cluster
initdb -D /tmp/pgdemo -A trust
pg_ctl -D /tmp/pgdemo -o "-p 54329 -k /tmp -c fsync=off" -w start
createdb -h /tmp -p 54329 demo
```

Save the schema above plus `INSERT INTO jobs (status) SELECT 'pending' FROM generate_series(1,20000);` as `setup.sql`, and each claim statement as its own file (`skip.sql`, `cas.sql`, `cas2.sql`, using `:client_id` for the owner). Then, resetting the table before each run:

```bash
psql -h /tmp -p 54329 demo -f setup.sql
pgbench -h /tmp -p 54329 -n -c 8 -j 8 -t 1000 -f skip.sql demo
psql -h /tmp -p 54329 demo -Atc "select count(*) from jobs where status='running'"
```

Things worth changing: raise `-c` to 32 and watch naive CAS get worse while SKIP LOCKED stays at 100%, or widen the random offset range in `cas2.sql` and watch the waste shrink.

## Common Pitfalls

- **Crashed workers strand jobs.** Both approaches leave a job in `running` forever if the worker dies. Store a lease (`locked_until timestamptz`) and let a reaper, or the claim query itself, take back expired claims.
- **Ignoring the row count.** With CAS, `UPDATE 0` *is* the result. If your driver code doesn't check affected rows, you'll process jobs you don't own.
- **Completing without proving ownership.** When a slow worker finishes after its lease expired, its `UPDATE … SET status = 'done'` can overwrite the new owner's progress. Complete with a CAS: `WHERE id = $1 AND owner = $2 AND version = $3`.
- **Locking and working in one long transaction.** It pins a connection, blocks vacuum from cleaning up, and a crash rolls everything back. Claim fast, commit, work outside the transaction.
- **Expecting FIFO from SKIP LOCKED.** It's "roughly in order". If a job must not start before an earlier one, you need a different design (per-key queues, or advisory locks).
- **No partial index.** Without an index on the pending rows, every claim scans a table that grows with finished jobs.

## Choosing Between Them

| The situation | Reach for | Because |
|---|---|---|
| Many workers draining a job queue | `SKIP LOCKED` | No contention on the head row, no retry loop |
| A user edits a record that others may edit too | CAS on `version` | You must act on *that* row and detect that it changed |
| A state transition that must happen once (`pending` → `paid`) | CAS on `status` | The `WHERE` clause is the guard, and `UPDATE 0` tells you someone beat you |
| Strict ordering per entity | Neither alone | Partition the work by key, or use advisory locks |
| Queue throughput matters and you also need safe completion | **Both** | `SKIP LOCKED` to claim, CAS on `owner` and `version` to finish |

The last row is the one most production queues end up at. They aren't rivals: SKIP LOCKED decides *who gets which row*, and CAS makes sure the *finish* is only accepted from the worker that still owns it.
