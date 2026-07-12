---
title: Introducing Overline
description: Record a browser workflow once, then run it again from a command palette — fully local, BYOK.
date: 2026-07-11
draft: true
---

Overline is a Chromium extension for AI-assisted browser macros. You describe an intent, demonstrate the workflow once, and Overline compiles a reusable script you can replay from `⌘⇧P` / `Ctrl⇧P`.

Recording uses your own LLM key. Playback is deterministic and does not call models.

## Draft notes

- Local-first: no accounts, no Overline servers, no analytics
- Pipeline: record → compile → sanitize → playback
- Open source: [reybahl/patch](https://github.com/reybahl/patch)

Replace this draft when ready to publish.
