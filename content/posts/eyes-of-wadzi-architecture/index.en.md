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

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false
---

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
Drive -- a tool she already uses daily. A scheduled pipeline detects changes,
downloads the images, generates the site, and deploys it. Zero interaction
with code, git, or the command line.

```
 PHOTOGRAPHER'S WORKFLOW            AUTOMATED PIPELINE
 ======================            ==================

 +----------------+
 |  Google Drive   |   Folders = galleries
 |                 |   Photos  = gallery items
 |  /galleries/    |   File descriptions = captions
 |    /spain/      |   Folder descriptions = body text
 |      img1.jpg   |
 |      img2.jpg   |
 |    /landscapes/  |
 |      ...        |
 +--------+--------+
          |
          | Google Drive API (read-only)
          v
 +--------+--------+
 |  sync_drive.py   |   Download images
 |                 |   Generate index.md per gallery
 |                 |   Generate _index.md for listing
 +--------+--------+
          |
          | content/galleries/ (generated)
          v
 +--------+--------+
 |  Hugo            |   Static site generator
 |                 |   Responsive image processing
 |                 |   Gallery theme (Pig.js layout)
 +--------+--------+
          |
          | public/ (static HTML + images)
          v
 +--------+--------+
 |  GitHub Pages    |   CDN-backed hosting
 |                 |   HTTPS by default
 |                 |   Free
 +-----------------+
```

## System Architecture

### Component Overview

```
+------------------------------------------------------------------+
|                     GitHub Actions Workflow                        |
|                                                                    |
|  +----------+     +----------+     +---------+     +----------+   |
|  |  check   |---->|  build   |---->| deploy  |---->| GitHub   |   |
|  |  (fast)  |     |  (full)  |     |         |     | Pages    |   |
|  +----------+     +----------+     +---------+     +----------+   |
|   Scheduled        On change        Always if                      |
|   only             or push          build OK                       |
+------------------------------------------------------------------+

Triggers:
  - push to main        --> skip check, always build + deploy
  - cron (every 5 min)  --> check first, build only if Drive changed
  - manual dispatch     --> skip check, always build + deploy
```

### Trigger Schedule

The cron schedule targets the hours the photographer is most likely editing:

```
UTC Hour:  06  07  08  09  ...  19  20  21  22
Local:     07  08  09  10  ...  20  21  22  23
            |---|---|---|         |---|---|---|
            Morning window       Evening window
            (every 5 min)        (every 5 min)
```

Outside these windows, only pushes or manual triggers cause builds.

### The Check Job (Lightweight)

Runs on schedule only. Purpose: avoid full builds when nothing changed.

```
check_drive_changes.py
======================

  Read last_sync.txt from cache
         |
         v
  +-- Has timestamp? --+
  |                     |
  No                   Yes
  |                     |
  v                     v
  "Changed"       Query Drive API:
  (first run)     "files modified after {timestamp}?"
                        |
                  +-----+-----+
                  |           |
                 Yes          No
                  |           |
                  v           v
              "Changed"   "No changes"
              (build)     (skip build)
```

The timestamp is persisted between runs via GitHub Actions cache.

### The Build Job (Full)

```
  Install Hugo + Dart Sass
         |
         v
  Checkout repo
         |
         v
  sync_drive.py          <-- Downloads ALL images from Drive
  |                          Generates content/galleries/
  |  For each subfolder:
  |    1. Create gallery dir
  |    2. Download images (sanitise filenames)
  |    3. Generate index.md with front matter
  |  Generate _index.md (gallery listing)
         |
         v
  Save sync timestamp    <-- For next check job
         |
         v
  Hugo build              <-- Processes images (thumbnails, srcsets)
  |                           Renders HTML from theme
  |                           Minifies output
         |
         v
  Upload artifact         <-- Static files in public/
```

### The Deploy Job

Deploys the artifact to GitHub Pages. Runs if and only if the build
job succeeded. Uses `always()` condition to handle the case where the
check job was skipped (push/manual triggers).

## Data Flow: Drive Folder to Web Page

```
Google Drive                    Generated Files                  Website
============                    ===============                  =======

Folder: "spain"                 content/galleries/spain/
  name: "spain"          --->     index.md:
  description: "Photos          ---
    from Barcelona"               title: Spain
  created: 2026-02-14             date: 2026-02-14
                                  resources:
  File: "sunset.jpg"              - src: sunset.jpg
    desc: "Beach sunset"  --->      title: "Beach sunset"      [lightbox
  File: "market.jpg"              - src: market.jpg             caption]
    desc: ""              --->    ---
                                                                /galleries/
                                  Photos from Barcelona         spain/
                                                                  |
                                                                  +-- grid of
                                                                      responsive
                                                                      thumbnails
```

### Filename Sanitisation

Drive filenames can contain spaces, uppercase, and special characters
(`Copy of Copy of IMG-20241116-WA0011.jpg`). The sync script normalises them:

```
Original (Drive)                      Saved as
================                      ========
Copy of Copy of IMG-20241116.jpg  --> copy-of-copy-of-img-20241116.jpg
My Photo (2).JPG                  --> my-photo-(2).jpg
sunset.jpg                        --> sunset.jpg
```

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

```
Theme module chain:
  galleriesdeluxe (multi-gallery wrapper)
    +-- gallerydeluxe (single gallery renderer, Pig.js grid)
    +-- hugo-mod-misc/common-partials (SEO, Open Graph)
```

### Theme Overrides

The project overrides specific theme files to add caption support:

```
THEME FILE                              OVERRIDE PURPOSE
==========                              ================
partials/gallerydeluxe/init.html        Add "title" field to image JSON
                                        so captions are available in JS

partials/galleriesdeluxe/header.html    Show flat gallery list in nav
                                        instead of category hierarchy

js/gallerydeluxe/src/index.js           Display caption in lightbox when
                                        image title differs from filename

scss/galleriesdeluxe/vars-custom.scss   Style the caption overlay
```

### Image Processing Pipeline

Hugo processes each source image into multiple sizes for responsive loading:

```
sunset.jpg (original, e.g. 4000x3000)
  |
  +-- 20px wide   (blurred placeholder, inline base64)
  +-- 100px wide  (tiny thumbnail)
  +-- 250px wide  (small thumbnail)
  +-- 500px wide  (medium, grid view)
  +-- full size   (lightbox view)
```

These are referenced in a JSON manifest that Pig.js uses to build the
justified grid layout.

## Security Model

```
+---------------------+     read-only      +------------------+
| Service Account     |<------------------>| Google Drive     |
| (no human login)    |  drive.readonly    | Shared Folder    |
+---------------------+  scope only        +------------------+
         |
         | JSON key stored as
         | GitHub Actions secret
         v
+---------------------+
| GitHub Actions      |  Secrets are:
| (ephemeral runner)  |  - encrypted at rest
|                     |  - masked in logs
|                     |  - scoped to repo
+---------------------+
         |
         | Static files only
         v
+---------------------+
| GitHub Pages        |  No server-side code
| (CDN)               |  No database
|                     |  No authentication surface
+---------------------+
```

**Key security properties:**
- Service account has **read-only** access to one shared folder
- No human credentials in the pipeline
- Credentials passed via environment variable, never written to disk in CI
- The deployed site is fully static -- no server, no API, no attack surface
- GitHub Pages provides HTTPS by default

## Caching Strategy

Four caches reduce build time and API usage:

```
CACHE                   KEY                        PURPOSE
=====                   ===                        =======
pip packages            requirements.txt hash      Skip pip install (~5s saved)
Hugo modules            go.sum hash                Skip module download (~6s saved)
Hugo image cache        Built-in (4320h TTL)       Skip image reprocessing
Sync timestamp          "drive-sync-timestamp"     Skip builds when Drive unchanged
```

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
