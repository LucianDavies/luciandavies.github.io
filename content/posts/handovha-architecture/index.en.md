---
title: "Handovha: A Verifiable Handover, Without Accounts"
subtitle: "Magic links, immutable signatures, and a $6/month droplet"
date: 2026-09-18T00:00:00+01:00
lastmod: 2026-09-18T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "The architecture behind Handovha, a passwordless tool for generating verifiable handover certificates — how it keeps a signed record trustworthy without accounts, and why the data model got simpler over time, not more complex."
license: ""
images: []

tags: ["architecture", "node.js", "postgres", "express", "system-design"]
categories: ["projects"]

featuredImage: "https://images.unsplash.com/photo-1450101499163-c8848c66ca85?w=1200&q=80"
featuredImagePreview: "https://images.unsplash.com/photo-1450101499163-c8848c66ca85?w=600&q=80"

hiddenFromHomePage: false
hiddenFromSearch: false
---

Memory is not a receipt. A landlord and a tenant walk through a flat together, agree it's fine, and shake hands. Six months later one of them remembers a scuff on the wall being there already; the other doesn't. A fleet manager hands a van to a driver with a full tank and no dents; a week later, someone disagrees about which of those was true at handover. None of this is fraud, usually — it's just that nobody wrote anything down at the one moment it mattered.

<!--more-->

**Handovha** is a small web tool that solves exactly that: two people, on one phone, at the moment an item or responsibility changes hands, produce a signed, timestamped, verifiable certificate — photos, both signatures, a QR code — with no app to install and no account to create beforehand.

## What I Built

The core loop is a four-step wizard: what's being handed over, where, and to/from whom, followed by both parties signing on the same device, one after the other. Completing it generates a PDF certificate with a QR code, emails it to both parties, and publishes it at a public, permanent verification URL.

The interesting engineering isn't the wizard — it's what has to be true underneath it for the certificate to actually be worth trusting:

- No password, ever — identity is a magic link sent to your email
- Once signed, a field can't quietly change without invalidating the signature that covered it
- The verification page has to work for a stranger with no account, while a draft-in-progress stays private
- None of this needed a second server, a queue, or a database beyond Postgres

## How It Works (The Short Version)

1. You start a handover, describe the item, add photos, and name both parties by email
2. Each party signs on the same device, one after the other
3. Once both signatures are in, a certificate PDF is generated, emailed to both parties, and published at a public link with a QR code
4. Anyone holding that link or QR can verify it — no login required

If you're curious about the mechanics — how a signature actually gets protected, and why the data model here is a rewrite that got *simpler* rather than a rewrite that got more features — the rest of this post goes into that.

---

# Technical Deep-Dive

## Identity without passwords

Handovha never stores a password. Signing in is a magic link: you give an email address, get a random 32-byte token by email, and clicking it creates your account on the spot if you don't already have one.

```
POST /login  { email }
  → random token, hashed, stored with a 15-minute expiry
  → emailed as a plain link

GET /auth/:token
  → token looked up by its hash, checked for expiry + single use
  → find-or-create user, mark the token consumed — same transaction
  → new session issued
```

Two details matter more than they look:

- **The response to `/login` never reveals whether an account already existed.** Same message either way. Otherwise the endpoint becomes a way to check who has an account here just by watching what comes back.
- **Consuming the token and creating the user happen in one transaction.** If those were two separate steps, a crash in between would leave a technically-unconsumed token that could be replayed to create a duplicate identity for the same email.

{{< figure src="trust-model.svg" alt="Two visibility rules, checked at two different layers, sharing one predicate" >}}

## A signature has to mean something

The actual hard problem in a tool like this isn't collecting a signature — a `<canvas>` and a PNG blob gets you that in an afternoon. It's making sure the signature still means what it meant *when it was drawn*, for as long as the certificate exists.

Concretely: what happens if someone signs as the "from" party, and *then* the creator goes back and swaps a photo, or changes the item description, or edits the other party's name? If nothing else happened, that signature would now be silently attached to a version of the record the signer never actually saw.

Handovha's answer is to treat a signature as scoped to the exact state of the fields it covers, and to void it the moment any of those fields change:

```sql
-- Editing any signed field (photo, party, location, description)
-- deletes the relevant signature(s):
DELETE FROM handover_signatures WHERE handover_id = $1 AND party = ANY($2);

-- If the handover had already reached "completed", it's rolled back —
-- the certificate that was issued no longer matches the record it
-- was supposed to attest to.
UPDATE handovers
SET status = 'awaiting_signatures', certificate_id = NULL, certificate_key = NULL, completed_at = NULL
WHERE id = $1 AND status = 'completed';
```

{{< figure src="certificate-lifecycle.svg" alt="Handover status lifecycle — draft, awaiting signatures, completed, and the edit-clears-signature loop" >}}

This is the one piece of business logic in the whole app that isn't optional. Everything else is UX; this is what makes the word "certificate" honest.

The certificate ID itself (`HC-XXXXXXXX`, random hex) is generated and checked for a collision on insert, retried up to five times against a unique constraint at the database level — a bare loop is simpler and just as correct as pre-checking for existence, since the database is the actual source of truth for uniqueness either way.

## Public by design, but not indexed

The verification page — `GET /certificates/:certificateId` — is the one route in the app that's intentionally open to anyone, no session required. That's not an oversight; it's the entire point. A certificate that only its creator can check isn't a certificate, it's a private note.

But "anyone with the link can see it" is a different guarantee from "this shows up in search results," and a certificate carries real names and ID numbers. So the route is explicitly excluded from `robots.txt` and the sitemap even though it answers any request:

```
Disallow: /dashboard
Disallow: /handovers
Disallow: /certificates
Disallow: /login
```

Everywhere else, visibility follows one rule, enforced at two separate layers because they're reachable independently: **you can see a handover if you created it, or if you're named as a party by email once it's completed.** The dashboard query and the authenticated file-serving route both encode that same predicate rather than one trusting the other:

```sql
-- Dashboard listing
WHERE created_by_user_id = $1
   OR (status = 'completed' AND (from_email = $2 OR to_email = $2))
```

```ts
// File serving (photos, signatures, the certificate PDF itself)
if (handover.status !== "completed" && (!req.user || !canView(handover, req.user))) {
  return res.status(404).end();
}
```

A completed certificate's photos and signatures are served without auth too, since they back the public verification page. A draft in progress is never public, regardless of who asks.

## The rewrite that removed code

Handovha didn't start as a single-purpose handover tool. It started from a generic multi-tenant SaaS starter — tenants, roles, memberships, invites, projects, Postgres row-level security enforcing tenant isolation. Reasonable defaults for "some SaaS product," and completely wrong for what this needed to be.

Once the actual shape of the product was clear — one person creates a record, two people sign it, anyone with the link verifies it — that whole layer was dead weight. Migration `011_handover_pivot.sql` drops it outright:

```sql
-- The product pivoted from a generic multi-tenant SaaS starter
-- (workspaces, roles, invites, projects) to a single-purpose handover
-- certificate generator with magic-link identity. None of the
-- tenant/role machinery applies anymore, so it's dropped outright
-- rather than left dormant.
DROP TABLE IF EXISTS project_grants, projects, role_permissions, permissions,
  activity_log, invites, memberships, password_resets, tenants CASCADE;
```

No feature flag, no "keep it around in case," no unused columns left behind to confuse the next migration. If a system's shape has genuinely changed, the old shape is a liability sitting in the schema, not an asset. Deleting a table you're certain is dead is a much smaller risk than the slow accumulation of things nobody's sure are safe to remove.

## Storage as an interface, not a decision

Uploaded photos, signature PNGs, and generated certificate PDFs currently live on the droplet's disk, on a mounted volume so they survive a redeploy. But nothing in the route handlers knows that:

```ts
export interface Storage {
  save(key: string, data: Buffer, contentType: string): Promise<void>;
  read(key: string): Promise<Buffer>;
  url(key: string): string;
}
```

`LocalDiskStorage` is the only implementation today. The day this needs to move to S3 or R2 — for redundancy, or because a single droplet's disk stops being enough — that's a new class behind the same three methods, not a rewrite of every place a file gets touched.

## Hosting: deliberately boring

There's no Kubernetes here, no managed queue, no serverless functions. Handovha runs as a single Node process under systemd, on one $6/month DigitalOcean droplet, behind nginx terminating TLS via certbot. Postgres runs on the same box. A deploy is `npm run build`, rsync the build output over, `npm ci --production`, restart the service, then poll `/ready` until it's healthy.

That droplet runs more than one app — a second, unrelated tool shares it — and both are described in one YAML manifest that the setup and deploy scripts both read, rather than each app carrying its own bespoke infrastructure.

For the traffic this actually gets, that's not a compromise; it's the appropriately-sized amount of infrastructure. The interesting failure modes here are about data integrity — a signature meaning what it says it means — not about scaling a fleet of pods. Spending engineering effort on the latter before the former is solid would have been optimizing the wrong thing.

## What I'd revisit

Signing is same-device and sequential today — both parties pass one phone back and forth. It works for the landlord-tenant, driver-handoff case this was built for first, but it doesn't cover a remote handover where the two parties aren't in the same room. That's the next real feature, not a fix: a second signing link, emailed to the other party, rather than a shared screen.

---

If you want to see it: **[handovha.com](https://handovha.com)**.
