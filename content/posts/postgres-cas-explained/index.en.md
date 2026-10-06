---
title: "Compare-and-Swap Updates in PostgreSQL, Explained"
subtitle: "Part 3 of 3: the optimistic way, and why it struggles as a queue"
date: 2026-10-06T00:02:00+01:00
lastmod: 2026-10-06T00:02:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "How compare-and-swap updates work in PostgreSQL under READ COMMITTED, why naive CAS wastes two thirds of claims on a job queue (2745 of 8000), how to fix it, and where CAS is the right tool: versioned edits, state transitions, safe completion."
license: ""
images: []

tags: ["postgresql", "concurrency", "optimistic-locking", "sql"]
categories: ["system-design"]

featuredImage: "/images/posts/postgres-cas-explained.png"
featuredImagePreview: "/images/posts/postgres-cas-explained.png"

hiddenFromHomePage: false
hiddenFromSearch: false
---

Compare-and-swap means: change this row, but only if it is still what I last saw. In SQL that's an `UPDATE` with a guard in the `WHERE` clause, and you check how many rows it touched.

<!--more-->

This is part 3 of 3, following the [overview](/posts/postgres-skip-locked-vs-cas/) and [SKIP LOCKED explained](/posts/postgres-skip-locked-explained/).

## How It Works

```sql
UPDATE jobs
SET status = 'running', owner = 'w1', version = version + 1
WHERE id = 42 AND status = 'pending';
```

`UPDATE 1` means you won. `UPDATE 0` means someone else got there first, and you retry or give up.

Under the default `READ COMMITTED` isolation, when two workers run this at once the second `UPDATE` blocks on the row lock held by the first. When the first commits, the second **re-evaluates its `WHERE` clause against the new row**, sees `status = 'running'`, and updates nothing. So CAS in Postgres is not lock-free. It is a short wait followed by a re-check, and the `WHERE` condition is the "compare".

## Where It Is the Right Tool

- **Versioned edits.** `WHERE id = 42 AND version = 7` stops two people overwriting each other's changes (the lost update problem).
- **One-time state transitions.** `WHERE id = 42 AND status = 'pending'` makes `pending` → `paid` happen exactly once, and `UPDATE 0` tells you someone beat you.
- **Safe completion** of a claimed job, below.

In all of these you know *which* row you mean. That's what a queue lacks.

## Why It Struggles as a Queue

The tempting single-statement claim:

```sql
UPDATE jobs SET status = 'running', owner = 'w1', version = version + 1
WHERE id = (SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1)
  AND status = 'pending';
```

Every worker's subquery picks the *same* head row. One wins. The rest wait, re-check, fail, and report `UPDATE 0`, though thousands of other pending rows exist.

In the [benchmark](/posts/postgres-skip-locked-vs-cas/#the-measured-result) (8 clients, 8,000 attempts) this claimed **2,745**. Two thirds of the attempts were wasted, and the run only looked fast because it was doing nothing.

## Fixing the Contention

Spread the workers out so they rarely target the same row:

```sql
\set off random(0,200)
SELECT id AS cid FROM jobs WHERE status = 'pending' ORDER BY id OFFSET :off LIMIT 1 \gset
UPDATE jobs SET status = 'running', owner = 'w' || :client_id, version = version + 1
WHERE id = :cid AND status = 'pending';
```

That claimed **7,799** of 8,000 (2.5% wasted). It works, but you've added a tuning knob (the offset range), a second round trip and a retry loop, to approximate what [SKIP LOCKED](/posts/postgres-skip-locked-explained/) does in one statement.

## Completing Safely

CAS earns its place at the *end* of a queue job. A slow worker whose lease expired must not overwrite the new owner:

```sql
UPDATE jobs
SET status = 'done', version = version + 1
WHERE id = $1 AND owner = $2 AND version = $3;
```

`UPDATE 0` means you no longer own the job, so discard your result.

## Pitfalls

- **Ignoring the row count.** `UPDATE 0` *is* the result. If your driver code doesn't check affected rows, you'll act on rows you don't own.
- **Unbounded retries.** Retry with a limit and jitter, or contention turns into a thundering herd.
- **Forgetting `READ COMMITTED` semantics.** Under `REPEATABLE READ`, a lost race raises a serialization error instead of returning `UPDATE 0`, so you must catch it and retry.

## Use It When

You know exactly which row you're changing and need to detect that it moved. For "give me any available row", use [SKIP LOCKED](/posts/postgres-skip-locked-explained/), and use both when you need fast claims *and* safe completion.
