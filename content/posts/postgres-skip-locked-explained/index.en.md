---
title: "PostgreSQL FOR UPDATE SKIP LOCKED, Explained"
subtitle: "Part 2 of 3: the pessimistic way to let many workers drain a queue"
date: 2026-10-06T00:01:00+01:00
lastmod: 2026-10-06T00:01:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "How FOR UPDATE SKIP LOCKED works in PostgreSQL, the single-statement claim pattern for job queues, why it never wastes an attempt, and the pitfalls: ordering, long transactions, crashed workers."
license: ""
images: []

tags: ["postgresql", "concurrency", "queues", "sql"]
categories: ["system-design"]

featuredImage: "/images/posts/postgres-skip-locked-explained.png"
featuredImagePreview: "/images/posts/postgres-skip-locked-explained.png"

hiddenFromHomePage: false
hiddenFromSearch: false
---

`FOR UPDATE` locks the rows a query returns. Add `SKIP LOCKED` and a row that another transaction has already locked is skipped instead of waited for.

<!--more-->

This is part 2 of 3, following the [overview](/posts/postgres-skip-locked-vs-cas/). Part 3 covers [CAS](/posts/postgres-cas-explained/).

## How It Works

Normally, `SELECT ... FOR UPDATE` on a locked row *waits* until the other transaction finishes. With `SKIP LOCKED`, the locked row is simply not in your result. Eight workers running the same query at once each get a different row.

## The Claim Pattern

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

One statement, one round trip. A returned row is yours. An empty result means no unlocked pending row exists.

## What It Costs

In the [benchmark](/posts/postgres-skip-locked-vs-cas/#the-measured-result) (8 clients, 8,000 attempts) it claimed **8,000 of 8,000**. No attempt was wasted, because no worker ever targets a row another worker holds. Adding workers adds throughput instead of contention.

## Pitfalls

- **Ordering is not guaranteed.** If worker A holds job 1 and worker B skips it to take job 2, job 2 can start first. Fine for independent jobs, wrong if order matters. For per-entity ordering, partition the work by key or use advisory locks.
- **It sees an inconsistent view on purpose.** The PostgreSQL docs say it suits queue-like tables and isn't suitable for general-purpose work. Don't use it to answer "what is the oldest pending row?".
- **Don't hold the lock while working.** Claim with a status change and commit straight away. A transaction held open for the whole job pins a connection and holds back vacuum, and a crash rolls the claim back.
- **Crashed workers strand jobs.** A job left in `running` stays there. Store a lease (`locked_until timestamptz`) and let the claim query, or a reaper, take back expired ones.
- **No partial index.** Without an index on the pending rows, every claim scans a table that grows with finished jobs.
- **Finishing without proving ownership.** A slow worker whose lease expired can overwrite the new owner's progress. Complete with a [CAS](/posts/postgres-cas-explained/#completing-safely): `WHERE id = $1 AND owner = $2 AND version = $3`.

## Use It When

Many workers drain a queue of independent jobs and you want the simplest thing that doesn't waste work. If you need to change one *specific* row safely, you want [CAS](/posts/postgres-cas-explained/) instead.
