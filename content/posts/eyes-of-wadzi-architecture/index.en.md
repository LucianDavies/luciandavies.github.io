---
title: "Eyes of Wadzi — Architecture & Design"
subtitle: "Turning Google Drive into a zero-code photography portfolio"
date: 2026-02-16T00:00:00+01:00
lastmod: 2026-02-16T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "How I built a free, zero-maintenance photography portfolio that uses Google Drive as a CMS, Hugo for static generation, and GitHub Actions for CI/CD."
license: ""
images: []

tags: ["architecture", "hugo", "google-drive", "github-actions", "python", "ci-cd"]
categories: ["projects"]

featuredImage: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&q=80"
featuredImagePreview: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=600&q=80"

hiddenFromHomePage: false
hiddenFromSearch: false
---

Someone close to me is a photographer. She needed a website to show her work — somewhere people could browse her photos, organised into galleries, without paying a monthly subscription.

The catch? She's not technical. At all. She doesn't want to open a terminal, learn a code editor, or think about "deploying" anything. She just wants to put photos in a folder and have them show up on a website.

<!--more-->

I looked at the usual options. **Squarespace** and **SmugMug** charge monthly fees and lock you into their platform. Self-hosted galleries need a server to maintain. Static site generators like Hugo are free, but they expect you to use git and the command line to publish — a non-starter for someone who just wants to drag and drop.

The gap was clear: nothing sat between **"I just want to drag photos into a folder"** and **"here's your website"**.

## What I Built

The answer turned out to be something she already uses every day: **Google Drive**.

She creates folders in Drive — one per gallery — and drops photos into them. That's it. That's her entire workflow. Behind the scenes, an automated pipeline checks Drive for changes every few minutes, downloads any new or updated photos, builds a website from them, and publishes it. She never touches code, never opens a terminal, and never runs a deploy command.

The website is completely free to host, loads fast, and looks good on any device. The whole thing costs exactly nothing to run.

## How It Works (The Short Version)

1. She organises photos into folders on Google Drive
2. Every few minutes, a scheduled job checks if anything changed
3. If it did, the pipeline downloads the photos, generates the site, and publishes it
4. The website updates itself — no human intervention needed

If you're curious about the nuts and bolts, the rest of this post goes deep into the architecture and technical decisions. If you're not, the summary above is really all there is to it.

---

# Technical Deep-Dive

## The Problem

A photographer needs an online portfolio. The requirements sound simple:

- Upload photos from a phone or laptop
- Organise them into galleries
- Have them appear on a website
- No terminal. No code editors. No deployment commands.

Most solutions fail on at least one count. Managed platforms (Squarespace,
SmugMug) cost money and lock you in. Self-hosted galleries need a server. Static
site generators need technical knowledge to publish. The gap is between **"I just
want to drag photos into a folder"** and **"here's your website"**.

## The Solution

Google Drive becomes the CMS. The photographer manages folders and photos in
Drive — a tool she already uses daily. A scheduled pipeline detects changes,
downloads the images, generates the site, and deploys it. Zero interaction
with code, git, or the command line.

{{< figure src="pipeline-overview.svg" alt="Pipeline overview — from Google Drive to GitHub Pages" >}}

## System Architecture

### Component Overview

{{< figure src="github-actions-workflow.svg" alt="GitHub Actions workflow — check, build, deploy pipeline" >}}

### Trigger Schedule

The cron schedule targets the hours the photographer is most likely editing:

{{< figure src="trigger-schedule.svg" alt="Trigger schedule — morning and evening windows" >}}

### The Check Job (Lightweight)

Runs on schedule only. Purpose: avoid full builds when nothing changed.

{{< figure src="check-job-flowchart.svg" alt="Check job flowchart — lightweight change detection" >}}

The timestamp is persisted between runs via GitHub Actions cache.

### The Build Job (Full)

{{< figure src="build-job-steps.svg" alt="Build job steps — from install to artifact upload" >}}

### The Deploy Job

Deploys the artifact to GitHub Pages. Runs if and only if the build
job succeeded. Uses `always()` condition to handle the case where the
check job was skipped (push/manual triggers).

## Data Flow: Drive Folder to Web Page

{{< figure src="data-flow.svg" alt="Data flow — from Drive folder to web page" >}}

### Filename Sanitisation

Drive filenames can contain spaces, uppercase, and special characters
(`Copy of Copy of IMG-20241116-WA0011.jpg`). The sync script normalises them:

| Original (Drive) | Saved as |
|---|---|
| `Copy of Copy of IMG-20241116.jpg` | `copy-of-copy-of-img-20241116.jpg` |
| `My Photo (2).JPG` | `my-photo-(2).jpg` |
| `sunset.jpg` | `sunset.jpg` |

Rule: lowercase, spaces to hyphens. This prevents URL encoding issues and
keeps Hugo's image processing happy.

## Repository Structure

```
eyes-of-wadzi/
|
+-- .github/workflows/
|     hugo.yml                 CI/CD pipeline (check -> build -> deploy)
|
+-- assets/
|     js/gallerydeluxe/src/
|       index.js               Lightbox caption logic (theme override)
|       helpers.js             Gallery helpers (copied from theme)
|       pig.js                 Justified image grid (copied from theme)
|     scss/galleriesdeluxe/
|       vars-custom.scss       Caption styling
|
+-- content/galleries/         GENERATED AT BUILD TIME (gitignored)
|     _index.md                Gallery listing page
|     spain/
|       index.md               Gallery front matter + body
|       sunset.jpg             Downloaded from Drive
|       market.jpg
|
+-- layouts/partials/
|     gallerydeluxe/
|       init.html              Adds "title" to image JSON (captions)
|     galleriesdeluxe/
|       header.html            Simplified nav (gallery list only)
|
+-- scripts/
|     sync_drive.py            Drive -> content/galleries/ generator
|     check_drive_changes.py   Lightweight change detection
|
+-- hugo.toml                  Site config (theme, image processing)
+-- requirements.txt           Python deps (google-api-python-client, google-auth)
+-- .gitignore                 Excludes content/galleries/
```

**Key point:** `content/galleries/` does not exist in git. It is entirely
generated at build time from Google Drive. The `.gitignore` ensures it is
never accidentally committed.

## Theme Architecture

The site uses the [Galleries Deluxe](https://github.com/bep/galleriesdeluxe)
Hugo theme, imported as a Hugo Module (not a git submodule).

| Module | Role |
|---|---|
| **galleriesdeluxe** | Multi-gallery wrapper |
| &emsp;gallerydeluxe | Single gallery renderer, Pig.js grid |
| &emsp;hugo-mod-misc/common-partials | SEO, Open Graph |

### Theme Overrides

The project overrides specific theme files to add caption support:

| Theme File | Override Purpose |
|---|---|
| `partials/gallerydeluxe/init.html` | Add "title" field to image JSON so captions are available in JS |
| `partials/galleriesdeluxe/header.html` | Show flat gallery list in nav instead of category hierarchy |
| `js/gallerydeluxe/src/index.js` | Display caption in lightbox when image title differs from filename |
| `scss/galleriesdeluxe/vars-custom.scss` | Style the caption overlay |

### Image Processing Pipeline

Hugo processes each source image into multiple sizes for responsive loading:

{{< figure src="image-processing.svg" alt="Image processing pipeline — responsive sizes from source" >}}

## Security Model

{{< figure src="security-model.svg" alt="Security model — service account to GitHub Pages" >}}

**Key security properties:**
- Service account has **read-only** access to one shared folder
- No human credentials in the pipeline
- Credentials passed via environment variable, never written to disk in CI
- The deployed site is fully static -- no server, no API, no attack surface
- GitHub Pages provides HTTPS by default

## Caching Strategy

Four caches reduce build time and API usage:

| Cache | Key | Purpose |
|---|---|---|
| pip packages | `requirements.txt` hash | Skip pip install (~5s saved) |
| Hugo modules | `go.sum` hash | Skip module download (~6s saved) |
| Hugo image cache | Built-in (4320h TTL) | Skip image reprocessing |
| Sync timestamp | `drive-sync-timestamp` | Skip builds when Drive unchanged |

The sync timestamp cache is the most impactful: it turns a ~40s build into
a ~10s no-op check on every 5-minute cron run when nothing has changed.

## Secrets & Configuration

Two GitHub Actions secrets are required:

| Secret                     | Content                              |
|---------------------------|--------------------------------------|
| `GOOGLE_CREDENTIALS_FILE` | Full JSON of service account key     |
| `GOOGLE_DRIVE_FOLDER_ID`  | ID of the shared Drive folder        |

The folder ID comes from the Drive URL:
`https://drive.google.com/drive/folders/<THIS_PART>`

## Cost

Everything in this stack is free:

| Component            | Cost    | Limits                              |
|---------------------|---------|-------------------------------------|
| Google Drive         | Free    | 15 GB storage                       |
| Google Drive API     | Free    | 20,000 queries/day                  |
| GitHub Actions       | Free    | 2,000 min/month (public repos)      |
| GitHub Pages         | Free    | 1 GB site size, 100 GB bandwidth/mo |
| Hugo                 | Free    | Open source                         |

At ~10s per check and 5-minute intervals over 7 hours/day, scheduled runs
consume roughly **7 x 12 x 10s / 60 = ~14 minutes/day** of Actions time.
