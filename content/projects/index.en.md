---
title: "Projects"
subtitle: "Things I've built"
date: 2026-09-19T00:00:00+01:00
lastmod: 2026-09-19T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "Projects by Tonderai Khatai — Handovha, Mail Koenig, and Eyes of Wadzi, with links to each live app and the architecture write-up behind it."
license: ""
images: []

tags: ["projects"]
categories: []

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: true
hiddenFromSearch: false
---

Small, self-hosted apps I've designed and built end to end — product, code, and the infrastructure underneath. Each one has a write-up on how it actually works.

<div style="display:flex;align-items:center;gap:.6rem;margin:1.5rem 0 .4rem;">
  <img src="/icons/handovha.png" alt="" width="28" height="28">
  <h2 style="margin:0;">Handovha</h2>
</div>

A passwordless tool for turning a physical handover into a signed, verifiable certificate — photos, both signatures, and a QR code, with no account required to check it. Magic-link identity, a signature that voids itself the moment the record it covers changes, and a public verification page that proves authenticity without exposing anyone's ID number to a stranger.

**Stack:** Node.js, TypeScript, Express, Postgres, PDFKit — a single DigitalOcean droplet.

[Visit handovha.com](https://handovha.com) · [Architecture write-up](/posts/handovha-architecture/) · [Why functional core, imperative shell fits it](/posts/functional-core-imperative-shell-micro-apps/)

---

<div style="display:flex;align-items:center;gap:.6rem;margin:1.5rem 0 .4rem;">
  <img src="/icons/mail-koenig.svg" alt="" width="28" height="28">
  <h2 style="margin:0;">Mail Koenig</h2>
</div>

An email-campaign tool built around a simple bet: a Postgres table with `FOR UPDATE SKIP LOCKED` is a completely adequate job queue at this scale, with no Redis or SQS in sight. Handles retries with jittered backoff, recovers cleanly from a worker crashing mid-batch, and keeps webhook delivery events idempotent against a provider that retries.

**Stack:** Node.js, TypeScript, Express, Postgres — shares a droplet with Handovha.

*Not publicly launched yet — still in active development.* [Architecture write-up](/posts/mail-koenig-architecture/)

---

<div style="display:flex;align-items:center;gap:.6rem;margin:1.5rem 0 .4rem;">
  <img src="/icons/eyes-of-wadzi.svg" alt="" width="28" height="28">
  <h2 style="margin:0;">Eyes of Wadzi</h2>
</div>

A zero-code photography portfolio for a photographer who doesn't want to touch a terminal. Google Drive is the entire CMS — drop photos in a folder, and a scheduled pipeline detects the change, rebuilds the site, and publishes it. Free to host, and needs no maintenance from either of us.

**Stack:** Hugo, Python, GitHub Actions, GitHub Pages — Google Drive as the CMS.

[Visit the site](https://coldsteam-studio.github.io/eyes-of-wadzi/) · [Architecture write-up](/posts/eyes-of-wadzi-architecture/)

---

<div style="display:flex;align-items:center;gap:.6rem;margin:1.5rem 0 .4rem;">
  <img src="/icons/digitalocean.png" alt="" width="28" height="28">
  <h2 style="margin:0;">The infrastructure behind these</h2>
</div>

Handovha and Mail Koenig both run on one shared DigitalOcean droplet, provisioned from a single YAML manifest and a handful of idempotent bash scripts — no Kubernetes, no Terraform, no PaaS.

[How it's set up](/posts/digitalocean-infra-setup/)
