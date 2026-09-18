---
title: "About"
subtitle: "Tonderai Khatai"
date: 2022-12-10T16:37:30+01:00
lastmod: 2022-12-10T16:37:30+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://myself@tldkhatai.com"
description: "About Tonderai Khatai — senior software engineer specialising in system design, distributed systems, and payments infrastructure in Node.js and TypeScript."
license: ""
images: []

tags: ["about", "communication", "personal brand"]
categories: []

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: true
hiddenFromSearch: false
---


## Hi, I'm Tonderai

I'm a software engineer with over a decade of experience, based in Berlin. I've spent that time building systems at companies including **Klarna** and **On** — both of which went public during my tenure. At Klarna, I'm a Senior Software Engineer on the Transaction Banking team, working on SEPA and DCL-based credit transfer and direct debit systems, merchant payout infrastructure, and real-time transaction notifications, built on AWS (SQS, Lambda) and Kafka — including leading migration efforts across these systems and mentoring engineers on design and code quality. At On, I was Technical Lead for 3rd-party logistics (3PL) and EDI integrations with Dynamics 365 ERP, leading and mentoring a team of three backend engineers — all three went on to lead teams of their own.

My work sits on the decisions that are expensive to get wrong: where service boundaries should live, how a system degrades under partial failure, what happens when an upstream payment rail misbehaves in production. The stack is mostly **Node.js** and **TypeScript** professionally; I also tinker with **Rust** on the side. The constant across all of it is reasoning about a system's failure modes before they become incidents.

### What this blog is about

This is where I write up the architecture and tradeoffs behind systems I've built — not tutorials, but the reasoning: why a design won out over its alternatives, what broke, what the failure model actually looked like, and what I'd change with hindsight. Recent examples: [Handovha](/posts/handovha-architecture/) on keeping a signature trustworthy without accounts, [Mail Koenig](/posts/mail-koenig-architecture/) on building a send queue out of nothing but Postgres, [why functional core, imperative shell fits micro apps](/posts/functional-core-imperative-shell-micro-apps/), and [Building a Zero-Code Photography Portfolio](/posts/eyes-of-wadzi-architecture/).

If a post helped you, or you want to talk systems design, feel free to reach out.

[LinkedIn](https://www.linkedin.com/in/tldkhatai/) | [GitHub](https://github.com/LucianDavies)
