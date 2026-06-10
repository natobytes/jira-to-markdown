# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A Jira Wiki markup → Markdown online converter that runs entirely in the browser. The whole app is a single self-contained `index.html` (CSS + vanilla JS, no dependencies, no build step, no framework).

## Development

There is no build, lint, or test tooling. To run, open `index.html` directly in a browser, or serve it:

```bash
python3 -m http.server  # then visit http://localhost:8000
```

Verify changes manually in the browser — the footer's "Load example" link exercises most Jira syntax (headings, bold/strike, nested lists, tables, quote/warning panels, code blocks).

## Architecture

All logic lives in one IIFE inside `index.html`. There are three independent converters plus wiring:

1. **`jiraToMd(src)`** — Jira wiki markup → Markdown. Line-based loop handling block constructs (`{code}`/`{noformat}` fences, `{quote}`/`{panel}`/`{info}`/etc. panels which recurse into `jiraToMd`, `||header||` tables, `h1.` headings, `*`/`#` lists, `bq.` quotes). Inline formatting is handled by `inline(text)`.
2. **`mdToHtml(md)`** — a tiny Markdown renderer used for both preview panes (the Jira-side "Preview" renders the *converted* Markdown, not Jira itself).
3. **`htmlToMd(html)`** — rich-paste path: when a paste carries a `text/html` clipboard flavor with real formatting (`htmlHasFormatting`), the HTML (e.g. copied straight from a Jira ticket) is converted to Markdown via DOMParser and recursive node rendering.

Key invariants:

- **Placeholder stashing**: `inline()` protects already-converted segments (code spans, links, strikethrough) with ``\u0000N\u0000`` placeholders (`stash`/`restore`, module-level `PH` array reset by `jiraToMd`) so later regex rules can't corrupt them. `mdInline()` uses the same trick with `\u0001` for code spans. Preserve this pattern when adding inline rules — rule order matters.
- **`inputIsMarkdown` flag**: after a rich paste the textarea already contains Markdown, so `run()` must skip `jiraToMd`. Typing into an empty box resets the flag. Breaking this double-converts pasted content.
- Rendering goes through `esc()` HTML-escaping before `mdInline()`; keep that ordering to avoid XSS via pasted content.
