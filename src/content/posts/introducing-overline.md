---
title: Introducing Overline
description: Generate a browser workflow using an LLM, then run it anytime from a command palette or keyboard shortcut. Fully local, BYOK.
date: 2026-07-11
draft: true
---

Overline is a Chromium extension for AI-assisted browser macros. You describe an intent, demonstrate the workflow once, and Overline compiles a reusable script you can replay from `⌘⇧P` / `Ctrl⇧P`.

Recording uses your own LLM key. Playback is deterministic and does not call models.

## Background

Some apps, like Linear or Visual Studio Code, are incredibly nice to use. As a power user, one of my favorite features is the deeply integrated keyboard shortcuts. I can open a file, search for an issue, or navigate to a line number all in a few keystrokes.

But, obviously, most apps don't have this kind of user experience. Most tools are designed for the mouse, and many common workflows involve clicking through a series of menus and buttons. If you're just using those tools once in a while, it's not a big deal. But if you, like me, spend hours a day in a tool, having to switch from keyboard mode to mouse mode might cause you to lose your flow state. Every time I push code from my local branch to the remote repository, I like to review the diff on GitHub (yes, I still read code) but the diff viewer is not very keyboard-friendly. I scroll with my keyboard but have to switch to the mouse to mark a file as viewed. Every time I make and push new changes, I need to click the refresh button that pops up (since `⌘R` makes the whole page reload). And then, finally, once I'm done reviewing all the files and marking each one as viewed, I click to go back to the conversation tab, click "Squash and merge", and then "Confirm merge". Even though this adds a few seconds, this adds up when you're doing it a dozen times a day, every day of the week.

I wanted to be able to use keyboard shortcuts - `Cmd Shift V` to mark a file as viewed, `Cmd Option R` to refresh the page, `Cmd Shift M` to merge a pull request, and so on.

Overline was born out of this desire: not just for GitHub, but for any tool across the web. It acts as a layer on top of the web apps you use, allowing you to generate "macros" using an LLM, and running them anytime from a command palette or keyboard shortcut.

