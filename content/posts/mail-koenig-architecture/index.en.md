---
title: "Mail Koenig: A Send Queue Made Entirely of Postgres"
subtitle: "No Redis, no SQS — FOR UPDATE SKIP LOCKED, a backoff curve, and a worker that can die mid-batch"
date: 2026-05-14T00:00:00+01:00
lastmod: 2026-05-14T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "The architecture behind Mail Koenig, a small email-campaign tool: how a Postgres table doubles as a job queue with FOR UPDATE SKIP LOCKED, how retries and crash recovery work without a queueing library, and how webhook delivery events stay idempotent against a provider that retries."
license: ""
images: []

tags: ["architecture", "postgres", "node.js", "system-design", "queues"]
categories: ["projects"]

featuredImage: "https://images.unsplash.com/photo-1596526131083-e8c633c948d2?w=1200&q=80"
featuredImagePreview: "https://images.unsplash.com/photo-1596526131083-e8c633c948d2?w=600&q=80"

hiddenFromHomePage: false
hiddenFromSearch: false
---

Sending an email campaign to a few thousand people looks simple until the third recipient's inbox provider rate-limits you mid-send, the fourth bounces permanently, and the process doing the sending gets killed by a deploy halfway through. **Mail Koenig** is a small campaign tool I built to send exactly that kind of batch — and the interesting part of it isn't the campaign editor, it's the fact that none of the above needed a message queue.

<!--more-->

There's no Redis, no SQS, no BullMQ. The entire send pipeline — queueing, claiming, retrying, recovering from a crash — is a handful of SQL statements against one Postgres table, run in a loop.

## What I Built

Mail Koenig lets someone import contacts, write a campaign, and send it from their own verified domain (or a shared one while they're getting started). A background worker process picks up campaigns marked for sending and works through their recipients in batches, calling Mailgun to actually deliver each batch and recording every delivery event — opened, clicked, bounced, complained — as it comes back over a webhook.

The part worth writing up is the worker: how a Postgres table plays the role a job queue normally would, without pretending to be one.

## How It Works (The Short Version)

1. Sending a campaign inserts one `campaign_recipients` row per eligible contact, `status = 'pending'`
2. A worker loop polls every few seconds: claim a batch of pending rows for a campaign, hand them to Mailgun as one batch-send call, mark the outcome
3. A failure gets classified as retryable or not; retryable ones get a backoff delay and another attempt, up to five
4. Mailgun's delivery webhooks (delivered, opened, bounced, complained, unsubscribed) update the same rows and, for anything suppression-worthy, the contact itself

If you want the mechanics — how a plain `UPDATE ... RETURNING` becomes a safe multi-worker queue, and what "safe" actually has to mean when the whole thing can be killed at any line — the rest of this goes into that.

---

# Technical Deep-Dive

## The claim query is the queue

There's no separate broker holding "jobs." A recipient row's `status` column *is* the job state, and claiming a batch to work on is one query:

```sql
UPDATE campaign_recipients cr
SET status = 'sending', locked_at = now(), locked_by = $1, attempts = attempts + 1
FROM contacts c
WHERE cr.contact_id = c.id
  AND cr.id IN (
    SELECT id FROM campaign_recipients
    WHERE campaign_id = $2
      AND status = 'pending'
      AND (next_attempt_at IS NULL OR next_attempt_at <= now())
    ORDER BY id
    FOR UPDATE SKIP LOCKED
    LIMIT $3
  )
RETURNING cr.id, cr.campaign_id, cr.contact_id, cr.email, cr.name, cr.attempts, c.unsubscribe_token
```

`FOR UPDATE SKIP LOCKED` is what makes this safe to run concurrently: if a second worker process ever runs this same query at the same moment, Postgres has each one skip rows the other has already locked instead of blocking on them or double-claiming them. Today there's exactly one worker process, so nothing is actually racing — but the query is correct for more than one without needing to become a different query later. The claim is scoped to a single campaign per call, too, so a batch never mixes recipients from two different campaigns: each Mailgun batch-send call needs one sender/domain, and mixing campaigns would mean mixing senders.

## Every batch outcome is one of three things

The worker loop itself is almost boring on purpose:

```typescript
for (;;) {
  try {
    const reclaimed = await reclaimStale(pool);
    await activateDueScheduledCampaigns(pool);
    const campaigns = await getSendingCampaigns(pool);
    for (const campaign of campaigns) {
      const claimed = await claimBatch(pool, campaign.id, BATCH_SIZE, workerId);
      if (claimed.length > 0) await sendBatch(pool, campaign, claimed);
      await maybeCompleteCampaign(pool, campaign.id);
    }
  } catch (err) {
    console.error("worker: loop iteration failed", err);
  }
  await sleep(POLL_MS);
}
```

There's no scheduler process either — "activate scheduled campaigns whose time has come" is one `UPDATE ... WHERE scheduled_at <= now()` in the same loop, not a cron job or a separate service.

`sendBatch` makes exactly one Mailgun API call per claimed batch (up to 500 recipients, under Mailgun's 1,000-recipient cap), and the whole batch shares one outcome: it succeeds, or the call throws and every recipient in it gets classified individually:

```typescript
export function isRetryable(err: any): boolean {
  const message: string = err?.message || "";
  if (NON_RETRYABLE_MESSAGE_PATTERNS.some((p) => p.test(message))) return false; // bounced/unsubscribed — will fail identically forever

  const status: number | undefined = err?.status ?? err?.statusCode;
  if (typeof status === "number") {
    if (status === 429) return true;   // rate limited
    if (status >= 500) return true;    // Mailgun-side error
    return false;                      // other 4xx — invalid request, won't succeed on retry
  }
  return true; // network/timeout/unknown — assume transient
}

export function backoffDelayMs(attempts: number): number {
  const base = 5_000;
  const max = 15 * 60_000;
  const exp = Math.min(base * 2 ** attempts, max);
  return exp + Math.random() * exp * 0.2; // jitter to avoid thundering herd
}
```

Getting this classification wrong in either direction is a real failure mode, not a nitpick: treating a permanent bounce as retryable means hammering a dead address five times for no reason; treating a rate limit as permanent means silently dropping mail that would have gone through a minute later. The rule reads as almost tautological once it's written down — retry what's transient, don't retry what will fail the same way again — but it has to be encoded somewhere, and here it's one small pure function instead of scattered `if` statements at each call site.

The jitter on the backoff matters at the batch level specifically: if several batches fail at once (a Mailgun-wide blip), a fixed exponential delay would bring them all back at exactly the same moment and immediately re-trip whatever caused the first failure. Spreading that by up to 20% is cheap insurance against re-creating the exact spike that just happened.

## A worker that gets killed mid-batch shouldn't lose the row

The failure mode that's easy to forget about: the worker process itself dies — OOM, a deploy restart, `kill -9` — *between* claiming a batch and recording what happened to it. Without a fix, those rows sit in `status = 'sending'` forever, locked by a worker ID that no longer exists, and the campaign never finishes.

```sql
UPDATE campaign_recipients
SET status = 'pending', locked_at = NULL, locked_by = NULL
WHERE status = 'sending' AND locked_at < now() - ($1 * interval '1 millisecond')
```

Run at the top of every loop iteration with a ten-minute timeout, this resets anything that's been "sending" for too long back to pending, to be claimed again. That's an honest at-least-once guarantee, not exactly-once: if the worker crashed *after* Mailgun actually accepted the batch but *before* the row got marked sent, reclaiming it means a rare duplicate send. That's the correct tradeoff here — a duplicate marketing email is an annoyance; a campaign that silently stalls at 40% sent because one process hiccuped is a support ticket. Which failure mode you're willing to tolerate is a real design decision, not a default you get for free, and it's worth being explicit about which one you picked and why.

## Eligibility is checked twice, on purpose

A campaign snapshots its recipient list once, at send time — but for a large campaign, sending can take a while, and a contact's status can change mid-send: they unsubscribe from a *different* campaign, or bounce somewhere else, in the minutes between when the batch was claimed and when it actually goes out. So eligibility gets a live re-check immediately before the Mailgun call, not just once at campaign creation:

```typescript
async function partitionByEligibility(pool: Pool, rows: ClaimedRecipient[]) {
  const contactIds = rows.map((r) => r.contact_id);
  const { rows: activeContacts } = await pool.query<{ id: number }>(
    "SELECT id FROM contacts WHERE id = ANY($1) AND status = 'active'",
    [contactIds]
  );
  const activeSet = new Set(activeContacts.map((c) => c.id));
  return {
    eligible: rows.filter((r) => activeSet.has(r.contact_id)),
    ineligible: rows.filter((r) => !activeSet.has(r.contact_id)),
  };
}
```

Anyone who dropped out of eligibility since being claimed is marked `skipped`, not sent to, and not retried. This is the same instinct as the claim query's `FOR UPDATE SKIP LOCKED`: don't trust that the world still looks the way it did when you first read it — check again right before the action that can't be undone.

## Webhooks retry, so recording an event has to be idempotent

Mailgun retries a webhook delivery whenever it doesn't get a prompt `200` back — meaning the same delivery event can arrive at this app more than once. Recording it twice would double-count opens and clicks and, worse, could re-trigger the suppression logic redundantly. The fix is a unique constraint doing the real work, not application-level "have I seen this before" bookkeeping:

```sql
INSERT INTO email_events (campaign_recipient_id, event_type, raw_payload, occurred_at, mailgun_event_id)
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT (mailgun_event_id) DO NOTHING
```

If the insert didn't actually insert anything (`rowCount === 0`), every side effect downstream — advancing `campaign_recipients.status`, suppressing the contact — is skipped too, inside the same transaction. A duplicate delivery becomes a genuine no-op, not "recorded twice but only acted on once" (which would still leave the event log wrong). And a bounce, complaint, or unsubscribe on *any* campaign suppresses that contact from every future one — a stronger guarantee than relying solely on Mailgun's own suppression list, and the actual enforcement point for "don't email someone who bounced."

## What I'd revisit

This whole design leans on there being one worker process. `FOR UPDATE SKIP LOCKED` means a second worker could join safely today with zero code changes — but nothing currently restarts a crashed worker automatically beyond the next deploy; it's `systemd`'s job to notice the process died and bring it back, and the ten-minute stale-lock reclaim is the backstop for whatever happens in the gap. For the volume this sends today, that's the right amount of infrastructure. If that changes, the honest next step isn't reaching for a message broker — it's just running a second worker process against the same table and letting the claim query do what it was already written to do.
