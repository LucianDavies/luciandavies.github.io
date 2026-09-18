---
title: "Functional Core, Imperative Shell Makes Sense for Micro Apps"
subtitle: "Not the whole framework — just the one habit worth keeping"
date: 2026-09-18T00:00:00+01:00
lastmod: 2026-09-18T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "Functional core, imperative shell is usually pitched at domain-heavy systems with full DDD tactical patterns. For a micro app, the full pattern is more structure than the problem deserves — but the core discipline pays for itself immediately. Illustrated with real code from Handovha."
license: ""
images: []

tags: ["architecture", "functional-programming", "ddd", "node.js", "system-design"]
categories: ["projects"]

featuredImage: "https://images.unsplash.com/photo-1544197150-b99a580bb7a8?w=1200&q=80"
featuredImagePreview: "https://images.unsplash.com/photo-1544197150-b99a580bb7a8?w=600&q=80"

hiddenFromHomePage: false
hiddenFromSearch: false
---

Functional core, imperative shell usually gets pitched as a full architecture: pure domain functions in one layer, all I/O pushed to the edges, everything wired together through explicit ports. Kacper Antman's [Functional Domain-Driven Design, Simplified](https://antman-does-software.com/functional-domain-driven-design-simplified) is a good, concrete version of that pitch — it adapts DDD's tactical patterns (aggregates, services, value objects) for a functional style: **derivers** as pure functions that take all their inputs explicitly and return a discriminated union of outcomes, **controllers** as the imperative code that orchestrates databases and external services around those derivers, and **invariants** as small pure predicates that check one business rule each.

<!--more-->

That's a good shape for a domain-heavy system — several bounded contexts, a team large enough that "which layer does this logic belong in" is a real coordination question. Most of what I build isn't that. It's a micro app: one developer, one deploy target, a domain small enough to hold in your head. [Handovha](/posts/handovha-architecture/) — a tool that turns a physical handover into a signed, verifiable certificate — is about 2,000 lines of TypeScript. Standing up separate deriver/controller/repository layers for that would be more structure than the problem has earned.

But the core discipline underneath the framework — **decide first, as a pure function; act second, as an effect** — earns its cost immediately, even here. Here's where that already shows up in Handovha's code without anyone deliberately imposing the pattern, and where skipping it actually cost something.

## Where it already happens, unforced

**Visibility is a pure predicate, not a query.** Whether a user can see or edit a given handover is answered by two small functions that take plain data and return a boolean — no database handle, no request object:

```ts
function canView(handover: Handover, user: { id: number; email: string }): boolean {
  return (
    handover.created_by_user_id === user.id ||
    handover.from_email === user.email ||
    handover.to_email === user.email
  );
}

function canEdit(handover: Handover, user: { id: number }): boolean {
  return handover.created_by_user_id === user.id;
}
```

This is exactly an "invariant" in Antman's terms, and it falls out naturally from just not wanting to write the same permission logic twice. It gets called from two unrelated places — the dashboard listing and the authenticated file-serving route — and because it's pure, there's no way for those two call sites to quietly drift into checking slightly different things. Testing it needs no database, no mocks, no setup: construct a `Handover` object, construct a user, assert on the boolean.

**The decision about what changed is separated from the effect of clearing it.** When someone edits a handover after a signature has been collected, that signature has to be invalidated — otherwise it would silently cover data the signer never saw. The route handler computes the *decision* as plain booleans before doing anything about it:

```ts
const fromChanged =
  (handover.from_name || "") !== body.fromName ||
  (handover.from_email || "") !== body.fromEmail ||
  (handover.from_id_number || "") !== (body.fromIdNumber || "");
const toChanged = /* ...same shape for the other party... */;

const changedParties: Array<"from" | "to"> = [
  ...(fromChanged ? (["from"] as const) : []),
  ...(toChanged ? (["to"] as const) : []),
];
if (changedParties.length > 0) await clearSignatures(handover.id, changedParties);
```

`fromChanged`, `toChanged`, and the resulting array are pure — same inputs, same output, every time. `clearSignatures` is the imperative shell: it takes that already-made decision and turns it into `DELETE` and `UPDATE` statements. The pure step never touches the database, and the impure step never has to re-derive what changed — it just acts on what it was told. Nobody had to name this "functional core, imperative shell" for it to be the natural way to write this correctly; keeping the decision and the effect as two separate steps is just how you avoid the bug where the query does slightly more or less than what was actually decided.

**ID generation splits format from persistence the same way.** A certificate ID is a random 8-hex-character string with a fixed shape, checked against a real uniqueness constraint in Postgres:

```ts
async function generateCertificateId(): Promise<string> {
  return "HC-" + crypto.randomBytes(4).toString("hex").toUpperCase();
}

let certificateId = await generateCertificateId();
for (let attempt = 0; attempt < 5; attempt++) {
  try {
    await query("UPDATE handovers SET certificate_id = $1, ... WHERE id = $2", [certificateId, handover.id]);
    break;
  } catch (err) {
    if (!isCertificateIdTaken(err) || attempt === 4) throw err;
    certificateId = await generateCertificateId();
  }
}
```

The *shape* of a valid ID (`^HC-[0-9A-F]{8}$`) is a pure fact, checked elsewhere with a plain regex on the public verification route. Whether a *specific* ID is actually free right now is not something a pure function can know — that's real, current state, and the database is the only honest source of truth for it. So the retry loop doesn't try to pre-check availability in application code (which would just be racing the database anyway); it does the impure thing, lets Postgres's unique constraint be the actual judge, and only re-derives a new pure candidate on the rare collision. Trying to force this into a "pure deriver" would have meant either lying about what's knowable in advance, or dragging a database read into something that's supposed to be pure.

## Where skipping it cost something

The certificate PDF generator is the one place this discipline breaks down, and it's a real cost, not a hypothetical one. `buildCertificatePdf` is supposed to be a rendering function — given a handover, its photos, and its signatures, lay out a PDF. But it starts by doing I/O:

```ts
async function buildCertificatePdf(verifyUrl: string, handover: Handover, photos, signatures): Promise<Buffer> {
  const qrDataUrl = await QRCode.toDataURL(verifyUrl, { margin: 0, errorCorrectionLevel: "H" });
  const signatureImages = new Map<string, Buffer>();
  for (const sig of signatures) {
    signatureImages.set(sig.party, await storage.read(sig.storage_key));
  }
  const photoBuffers = await Promise.all(photos.map((p) => storage.read(p.storage_key)));
  // ...300 more lines of layout, now depending on storage having succeeded
}
```

Because the disk reads are interleaved with the layout logic instead of happening before it, there's no way to unit-test "does this certificate lay out correctly" without also standing up (or mocking) `storage`. A layout bug and a storage bug show up as the same kind of failure — a thrown exception somewhere inside one large async function — and the only way to tell them apart is to read the stack trace carefully. The fix is exactly the split this post has been describing: an imperative step that resolves `signatures` and `photos` into actual buffers first, handed to a pure `layoutCertificate(handover, photoBuffers, signatureBuffers, qrBuffer): PDFDocument` that never awaits anything. I haven't made that change yet — it's on the list — but writing this post is what made the shape of the fix obvious. That's usually how it goes: the pattern doesn't just prevent bugs, it makes the next refactor legible.

## Why the full pattern would be overkill here

Antman's framework earns its layers when the payoff is coordination: a `deriver` that several `controllers` can safely reuse because its type signature is the only contract that matters, repositories that let the same domain logic run against a real database in production and an in-memory fake in tests, aggregates whose invariants need enforcing across many entry points written by many people. Handovha has one entry point per operation, one person deciding where logic lives, and a domain small enough that "which layer is this in" is never actually confusing enough to need answering by folder structure.

Building out `src/derivers/`, `src/controllers/`, and `src/repositories/` for a codebase this size would mean more files to open to trace a single request, more indirection between "the button was clicked" and "here's the line that decided what happens," for a coordination problem that doesn't exist yet. The complexity budget of a micro app is better spent elsewhere.

What transfers regardless of scale is the discipline, not the ceremony: before you write an effect, write down the decision as something you could test without the effect. `canView`, the changed-party booleans, and the ID-format check all do that for free, as an artifact of writing small functions with obvious inputs — not because anyone set out to build a "functional core." The PDF generator didn't, and the cost showed up exactly where the pattern says it would: in the one place a pure decision and a real side effect got tangled into the same function.
