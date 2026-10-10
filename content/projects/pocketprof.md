---
title: "PocketProf"
date: 2026-10-09
tags: ["ai", "go", "react", "sqlite", "product-engineering"]
description: "An AI tutor that quizzes you before it teaches you, for any conceptual topic."
summary: "A personal AI tutor for any conceptual topic. It probes what you already know, plans a lesson around the gaps, then teaches one idea at a time."
showToc: false
disableAnchoredHeadings: false
---

**[askpocketprof.com](https://askpocketprof.com/?utm_source=portfolio&utm_medium=referral)**

PocketProf teaches you a topic (maths, science, history, economics, how systems work) by working out where your knowledge stops and starting there. Most AI learning tools explain what you already know and skip what you don't. PocketProf asks first.

## How it works

1. **Probe.** A short interview finds the edge of what you know and surfaces misconceptions.
2. **Plan.** It proposes a lesson plan built from your answers. Ideas you've shown you know are skipped, shaky ones get time, and misconceptions are taken apart first.
3. **Teach.** One idea at a time, with diagrams and pictures when they help. It explains directly when that's quicker and asks when working it out yourself will make it stick.
4. **Remember.** Every answer updates a per-user knowledge map, and spaced review brings ideas back. The next session on a subject starts where the last one stopped.

## Under the hood

- Go backend with a background worker, SQLite for storage, and a React frontend.
- Every change the browser sees is an event; the UI renders only from events plus a snapshot endpoint.
- Answer keys never leave the server before a card is answered.
- Google sign-in, per-user data scoping, and a per-user credit ledger that meters LLM usage.
- Subscription and top-up billing through Dodo Payments.
- Local, in-image embeddings for cross-subject linking, so there's no paid embedding API.
- Runs as a single Docker Compose stack.

## Stack

Go · React · SQLite · Docker · OpenRouter
