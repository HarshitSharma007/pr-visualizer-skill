# PR Visualizer — Skill

An agent skill that turns a GitHub PR link into a plain-language, learning-focused slide deck that makes the PR easy to review.

**Input:** a PR URL. **Output:** a slide deck (HTML first, PowerPoint second) — one slide per file change, plus an app-level slide showing what feature is changing, with a demo of the app itself.

## What a deck looks like

- **Cover** — the PR in one line, with the real numbers (files, +/−, triage counts)
- **App-level feature slide** — what feature is changing, with a demo of the app itself (labelled mock or real screenshot), Before/After toggle
- **HLD slide** — the feature's architecture, before → after
- **One slide per file** — header (`PR <number> / <n> of <total>`, filename, full path), numbered group, yellow **WHY this file changes** box, **INPUT → THIS FILE → RESULT** flow, pink **KEEP THIS DISTINCTION** box, and a source footer with the commit hash
- **Gates & tests slide** — safety gates (secrets, auth, validation, migrations, breaking changes, dependencies) and test coverage, with honest provenance

Style: **Classic Sketchbook** — light, warm cream paper, pale pastel boxes, hand-drawn fonts (Excalifont / Virgil), thin slate borders. No dark mode, no saturated colours.

## Install

Copy this folder into your skills directory, e.g.:

```sh
# Claude Code
cp -r pr-visualizer-skill ~/.claude/skills/pr-visualizer

# Muse (custom skills)
cp -r pr-visualizer-skill ~/workspace/custom-skills/pr-visualizer
```

Or add it with a skills manager:

```sh
npx skills add <your-github-username>/pr-visualizer-skill
```

## Use

Ask your agent to explain a PR, for example:

> Explain https://github.com/ahmedkhaleel2004/gitdiagram/pull/205 as a PR Visualizer deck

The skill fetches the real diff from the GitHub API (no token needed for public repos), triages every file into core / supporting / mechanical, finds the changed functions and their callers from the parsed code, checks the safety gates, and builds the deck — grounding every claim in a `file:line` reference.

## Example

A sample deck built with this skill: **gitdiagram PR #205, "Tighten the mobile repo card layout"** — 7 slides, cover + interactive phone app-demo + HLD + one slide per file (`main-card.tsx`, `copy-button.tsx`, `export-dropdown.tsx`) + gates & tests. All numbers grounded in the real diff and PR body (closed card 376px → 278px at 390px; 282 tests reported in the PR validation, not re-run).

## Rules the skill enforces

- **Ground everything in the real diff.** No invented functions, callers, numbers, or test results.
- **Plain language, key points only.** ~120 words max per file card.
- **Parsed edges are facts; guessed edges are suggestions.** Unresolvable calls go in a "Could not trace (dynamic)" bucket, never silently dropped.
- **Test presence is never presented as coverage %.**
- **The deck explains; it does not approve or block.**

## Companion project

The CLI that powers the fetching and triage steps (M1: PR → file list with triage) lives in the [PR Visualizer](https://github.com/) project — see `pr-visualizer` (TypeScript, MIT). This skill is the format and workflow; the CLI is the deterministic engine underneath it.

## License

MIT — see [LICENSE](LICENSE).
