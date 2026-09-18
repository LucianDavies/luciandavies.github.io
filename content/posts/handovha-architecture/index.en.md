---
title: "Handovha: A Verifiable Handover, Without Accounts"
subtitle: "Magic links, immutable signatures, and a $6/month droplet"
date: 2026-03-18T00:00:00+01:00
lastmod: 2026-09-18T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "The architecture behind Handovha, a passwordless tool for generating verifiable handover certificates — how it keeps a signed record trustworthy without accounts, using magic-link identity, signature invalidation, and a public-but-unindexed verification page."
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

If you're curious about the mechanics — how a signature actually gets protected, and what "public but verifiable" means in practice — the rest of this post goes into that.

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

## Public by design, but not everything is public

The verification page — `GET /certificates/:certificateId` — is the one route in the app that's intentionally open to anyone, no session required. That's not an oversight; it's the entire point. A certificate that only its creator can check isn't a certificate, it's a private note.

But "anyone with the link can verify this happened" is a narrower claim than "anyone with the link can see everything on it." A stranger who finds a certificate link shouldn't be able to read out someone's ID number or download their signature image — they just need to be convinced the record is genuine. So a non-party viewer gets a name, a date, and a *hash* standing in for anything actually personal:

```typescript
const isParty = !!req.user && (req.user.email === handover.from_email || req.user.email === handover.to_email);
const pdfUrl = isParty && handover.certificate_key ? storage.url(handover.certificate_key) : null;

let signatureHashes: Record<string, { hash: string; signedAt: Date }> | null = null;
if (!isParty) {
  signatureHashes = {};
  for (const sig of signatures) {
    const bytes = await storage.read(sig.storage_key);
    signatureHashes[sig.party] = { hash: hashPreview(bytes), signedAt: sig.signed_at };
  }
}
```

The hash is a truncated SHA-256 of the actual ID number or signature image bytes — deterministic, so the same underlying value always produces the same hash, but not reversible back into that value. That gets a non-party viewer something genuinely useful (if the record is later disputed, two people can compare hashes and confirm they're looking at the same signature without either of them re-exposing it) without handing out the thing itself. The signed party still sees everything in the clear — `isParty` is the only branch, not a separate role — because ID numbers and signature images are exactly what *they* need the certificate to actually contain.

{{< figure src="trust-model.svg" alt="Public verification shows a party everything and a stranger a tamper-evident hash instead" >}}

The file-serving route enforces the same split. Photos stay open on a completed handover, since they back the public page and were never the sensitive part; signatures and the PDF (which embeds both parties' ID numbers and full signature images) are restricted to the creator-or-party audience even after completion, closing off the obvious workaround of just fetching the file directly instead of the hashed page:

```typescript
const PARTY_ONLY_TYPES = new Set(["signatures", "documents"]);
// ...
const restrictedType = PARTY_ONLY_TYPES.has(req.params.type);
if ((handover.status !== "completed" || restrictedType) && (!req.user || !canView(handover, req.user))) {
  return res.status(404).end();
}
```

The certificate still carries real names, so the route stays out of the crawl entirely — "verifiable if you have the link" and "indexable" are still two different things:

```
Disallow: /dashboard
Disallow: /handovers
Disallow: /certificates
Disallow: /login
```

Everywhere else in the app, visibility follows one rule: **you can see a handover if you created it, or if you're named as a party by email once it's completed.** The dashboard listing encodes that same predicate independently, rather than trusting the file-serving route to be the only place it's enforced:

```sql
WHERE created_by_user_id = $1
   OR (status = 'completed' AND (from_email = $2 OR to_email = $2))
```

A draft still in progress is never public, regardless of who asks — that part hasn't changed; what changed is how much of a *completed* certificate a non-party actually gets to see.

## Storage as an interface, not a decision

Uploaded photos, signature PNGs, and generated certificate PDFs currently live on the droplet's disk, on a mounted volume so they survive a redeploy. But nothing in the route handlers knows that:

```typescript
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
