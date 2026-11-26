---
name: pr-visualizer
description: Turn a GitHub PR link into a plain-language slide deck — one slide per file change plus an app-level feature slide with an app demo. Use when someone shares a PR URL and asks what it changes, wants it explained file by file, or wants a shareable PowerPoint or HTML explainer.
---

# PR Visualizer

Turn a pull request into a deck a reviewer can understand in minutes — and learn from.

**Input:** a GitHub PR URL, e.g. `https://github.com/<owner>/<repo>/pull/<number>`.
**Output:** a slide deck — PowerPoint first, HTML as the interactive twin — one slide per file change, plus an app-level slide showing what feature is changing, with a demo of the app itself.

## Golden rules

1. **Ground everything in the real diff.** Fetch the PR first. Never invent functions, callers, numbers, or test results. If you cannot verify a claim from the diff, the PR body, or the repo, say so on the slide.
2. **Plain language, no jargon.** Explain like a senior engineer guiding a capable developer who does not know this codebase. Lead with the answer. Define a project-specific term the first time it matters, in one short phrase. Keep standard terms (API, cache, migration, queue) as they are.
3. **Key points only.** Every file card: what the file does (1 sentence), what changed (≤5 bullets), why it matters (1 sentence), watch out for (only if a gate fired). No card over ~120 words.
4. **Every claim pins to evidence.** Every function, edge, and gate finding carries a `file:line` reference. Every slide footer carries the source commit hash.
5. **The deck explains; it does not approve or block.** It never posts a review comment as a reviewer. It links to the diff and to the team's existing reviewer.

## Workflow

### Step 1 — Fetch the PR

Parse the URL into owner, repo, number (tolerate trailing slash, `/files`, query strings).

**Preferred method — the companion CLI (M1).** If the `pr-visualizer` CLI is available, use it; it already implements this step with triage:

```sh
prviz https://github.com/<owner>/<repo>/pull/<number> [--json]
```

It prints the PR title, author, base → head, and every file with additions/deletions, status, and triage bucket (core / supporting / mechanical). No token is needed for public repos; it honours `GITHUB_TOKEN` when set.

**Fallback — GitHub REST API directly.** No token is needed for public repos (60 requests/hour unauthenticated); honour `GITHUB_TOKEN` when set (`Authorization: Bearer $GITHUB_TOKEN`). Paginate `listFiles` (100/page).

```sh
# PR metadata (title, body, author, base/head SHAs, state)
curl -s https://api.github.com/repos/<owner>/<repo>/pulls/<number>

# Changed files (filename, status, patch, additions, deletions) — paginate with ?page=N
curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<number>/files?per_page=100&page=1"

# Raw unified diff
curl -s -H "Accept: application/vnd.github.v3.diff" \
  https://api.github.com/repos/<owner>/<repo>/pulls/<number>
```

Also fetch the full base + head contents of changed files (`GET /repos/{owner}/{repo}/contents/{path}?ref={sha}`) — the diff alone is not enough to know what a changed function calls.

Filter noise before any explanation: lockfiles, generated code, vendored code, images, `.map` files, formatting-only files.

If the PR is private and you have no token, stop and say so — do not guess at its contents.

### Step 2 — Triage every file: core / supporting / mechanical

Triage **before** reading closely, in this order:

- **Mechanical** — lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Pipfile.lock`, `poetry.lock`, `Gemfile.lock`, `go.sum`), generated code, vendored code, images, `.map` files, snapshots, binaries. Verify they are what they claim, then move on. A "rename-only" file with a logic edit hiding inside is a classic trap.
- **Supporting** — test files, `*.d.ts`, docs, config, pure renames (0 additions / 0 deletions), small changes (≤10 lines) in logic files. Scan for surprises.
- **Core** — everything else with real logic, especially files touching auth, state, money/payments, migrations, schema, tokens, or concurrency. Sensitive-path keywords (`auth`, `payment`, `migration`, `schema`, `token`, `price`, …) make a file core at any size — match whole path segments (e.g. `src/auth/`), not substrings: `tokenizer` and `pricing.md` do not count. More than 10 changed lines in a logic file is core.

**Grouping files into numbered groups.** A slide's group is the change's purpose, not a filename. Group files that serve one purpose (same feature area, directory, or commit) and name the group from that purpose, e.g. `1. Ribbon wiring` groups the gate declaration with the files that register behind it. Rules: order groups core-first in review order; number them `1.`, `2.`, …; keep at most 5 groups; give each group a short verb-first name; mechanical files get no group — they are listed in an appendix slide, not given their own slides. If a file fits two groups, put it in the group where its change matters most and mention the other group in its RESULT box.

Show the size up front: total files, total +/−, and a warning when the PR is over the 400-line review threshold — defect detection collapses above ~400 lines / 60–90 minutes, so say so and segment by commit or module.

### Step 3 — Changed functions (per file)

Parse the base and head versions of each changed file into an AST (tree-sitter; start with TS/JS + Python). Extract functions/classes with name, signature, and line ranges, and intersect those ranges with the changed-line ranges from the diff.

- Range overlaps an added/changed line → **changed**
- In head but not base → **added**
- In base but not head → **removed**

Store per function: qualified name, file, start/end lines (base + head), signature, status. This is the unit the file slides are built from. Never list a function the diff does not touch.

**Setup.** Install the tree-sitter grammars you need before parsing, e.g. `npm install web-tree-sitter` plus the `tree-sitter-typescript` / `tree-sitter-python` grammar packages, or use a `tree-sitter` CLI with grammars installed. **Fallback:** if no grammar exists for a file's language, do not guess functions from the text — use the hunk headers (`@@ ... @@` lines, which often name the enclosing function) and mark every function from that file `inferred — verify` on the slide.

### Step 4 — Callers: who calls this function, and from where

Build the call graph from the parsed AST — **parsed edges are facts; guessed edges are suggestions.** Never ask a model "who calls X?" as the source of truth; use it only to narrate the parsed graph.

For each added/changed function: direct callers (1 hop, each with `file:line`), then transitive callers (BFS, cap depth 3, cap 20 nodes — label the cap), plus what the function itself calls (callees, 1 hop).

Show unresolvable calls honestly in a **"Could not trace (dynamic)"** bucket — dynamic dispatch, dependency injection, route decorators, monkey-patching, and untyped `obj.method()` cannot be resolved statically. Never silently drop them, never invent an edge.

### Step 5 — HLD diagram of the feature

Use the C4 model as the scaffold: System Context → Container → Component. Draw the **feature's slice only** — entry point (user / API / job) → the changed components (highlighted) → the unchanged neighbours they touch → data stores / external systems. Before/after toggle with one shared layout: only the delta changes.

Cap at ~15 nodes; beyond that, group into containers and let the reader drill down. Every node maps to a real path; every edge is backed by an import/call edge or flagged `inferred — verify`.

**Layout — deterministic, never model-placed.** Compute positions with a layered layout (Dagre/ELK) or a fixed grid, then convert the laid-out graph to diagram elements. Rules: minimum node size 120×60px, 20–30px gaps, diagram text ≥14px, routes are orthogonal (rounded right-angle bends), arrows prefer the side facing their target and never pass through boxes or titles. A minimal Excalidraw element skeleton (fill in `x`, `y`, `id`, `seed` from the layout; text is a separate element bound to its container):

```json
{ "type": "rectangle", "x": 0, "y": 0, "width": 200, "height": 80,
  "strokeColor": "#3d4a5f", "backgroundColor": "#eaf2fe",
  "fillStyle": "solid", "strokeWidth": 1.6, "roughness": 1, "seed": 42 }
```

### Step 6 — Safety gates (deterministic checks, not vibes)

Run deterministic checks first; explain a finding in plain language, never invent one. Present as a panel: ✅ pass / ⚠️ check / 🔴 blocker, each with tool, rule, and `file:line` evidence. Only findings on changed lines can block; pre-existing issues are reported separately.

**Setup.** Install the scanners before the run: `pip install semgrep` (or `brew install semgrep`), `brew install gitleaks` (or download the release binary). Run Semgrep diff-aware so it only flags code the PR touches: `SEMGREP_BASELINE_REF=<base-sha> semgrep --config auto`. Run gitleaks on the change: `gitleaks detect --source . --verbose`. **Fallback:** if a tool is not installed, run the checks you can and mark the missing gate on the slide as `not run — <tool> missing`. Never invent a finding to fill an empty panel, and never report a gate as "pass" when it did not run.

| Gate | How to detect |
|---|---|
| Secrets / credentials | Scan added (`+`) lines for high-entropy strings, `api_key`, `secret`, `password`, `token`, `BEGIN (RSA\|EC\|OPENSSH)`, connection strings, `.env` changes. Tools: gitleaks, TruffleHog. A secret ever committed is compromised even if deleted. |
| Auth / authz on new endpoints | New route/handler with no auth middleware or guard in its call chain. Every new endpoint needs an *authorization* check, not just authentication; check IDOR (a user can only access resources they own). |
| Input validation / injection | User input reaching SQL / shell / HTML without validation; new query built by string concatenation; missing validation on new request fields. Tool: Semgrep, diff-aware (only flag code the PR touches). |
| Error handling | Empty `catch {}` / `except: pass`, new external call without timeout/retry, error messages leaking internals, swallowed errors in new code paths. |
| Migrations / schema | Files under `db/migrate`, `migrations/`, Alembic, Flyway, `schema.prisma` changes. Flag: non-null column with default on a big table, missing `CREATE INDEX CONCURRENTLY`, unbatched backfills, irreversible migration with no rollback note. |
| Breaking changes | Removed/renamed exported symbols, changed function signatures, removed/renamed API fields or routes, changed env-var or config keys. Compare exported symbols base vs head; suppress renames with `git diff --find-renames`. |
| Dependencies | New entries in `package.json` / `go.mod` / `pyproject.toml` / `Gemfile`. Check CVEs and licence fit; a new dependency is a new supply-chain surface. |

### Step 7 — Has it been tested?

Report, in this order:

1. Test files added/changed in the PR (by naming: `auth.ts` → `auth.test.ts` / `auth.spec.ts` / `__tests__/auth.ts`).
2. Test files that *import* the changed source files (naming + imports catches most real test relationships in practice — treat it as a heuristic, not a guarantee).
3. Changed functions with **no** covering test — the test-gap finding.
4. CI status (tests green/red on head).
5. Patch coverage % **only if a coverage report exists** (Codecov `codecov/patch` = only the lines the PR changed). Without a coverage report, label it honestly: *"No coverage report found in CI — showing test presence, not coverage."* Never present test presence as a coverage %. Coverage shows which lines ran, not whether the tests check anything.

## Deck structure

Slides, in this order. Navigate with Prev/Next, a clickable slide rail, and arrow keys.

1. **Cover** — PR number, title, author, base → head, one-line intent ("what changes for the user/system"), the real numbers (files, +/−, lines changed), triage counts, and a link to the live PR.
2. **App-level feature slide** — what feature is getting changed, in one line, with a **demo of the app itself** (see "Making the app demo" below). Before/After toggle where it helps (gate ON vs OFF, before vs after layout). Label any mock as a mock — never pass a mock off as a screenshot.
3. **HLD slide** — the feature diagram from Step 5, before → after.
4. **One slide per file change** — in triage order (core first, mechanical collapsed or last). See the per-file layout below.
5. **Gates & tests slide** — the gate panel from Step 6, the test list and test gaps from Step 7, and the size warning if the PR is over 400 lines.

### Making the app demo (slide 2)

Try these in order; use the first one that works, and say on the slide which one it is.

1. **Real screenshot (best).** Run the app locally (dev server or production build), open the changed screen at the viewport the PR cares about (e.g. 390px wide for a mobile change), and capture before (base) and after (head) with Playwright: `npx playwright screenshot --viewport-size=390,844 <url> after.png`. Label it "Screenshot — <viewport>, after this PR".
2. **Labelled mock.** If you cannot run the app, recreate the UI chrome (tabs, ribbon, panes, buttons) in HTML/CSS from the repo's own screenshots or docs, and label it clearly on the slide: "Mock — recreated UI, not a screenshot". Keep it faithful to the real layout; do not invent controls the app does not have.
3. **Diagram only (last resort).** If neither is possible, use a before/after diagram instead and write on the slide: "No app demo available — diagram only".

Never present a mock as a screenshot, and never draw a fake UI without the mock label.

## Per-file slide layout (exact)

Every file slide follows this layout, no exceptions:

```
PR <number> / <n> of <total files>
<FileName.ext>
<full/path/to/FileName.ext>

<group number>. <Group name>          e.g. "1. Ribbon wiring"

┌─ WHY this file changes ─────────────────────────────┐
│ One or two plain sentences. Why does this behaviour │
│ need its own change in this file?                   │
└─────────────────────────────────────────────────────┘

┌─ INPUT ────────┐    ┌─ THIS FILE ────────┐    ┌─ RESULT ────────┐
│ What feeds     │ →  │ What this file     │ →  │ What is true    │
│ into it.       │    │ actually does in   │    │ after it runs.  │
│                │    │ this PR.           │    │ Who checks the  │
│                │    │ Name the function  │    │ result, and     │
│                │    │ or value added.    │    │ where.          │
└────────────────┘    └────────────────────┘    └─────────────────┘

┌─ KEEP THIS DISTINCTION ─────────────────────────────┐
│ The one thing people usually get wrong about this   │
│ change. What this change does NOT do.               │
└─────────────────────────────────────────────────────┘

Source: <head commit sha, short> | High-level explanation, not proof of backend deployment
```

- **WHY box:** the reason, not a repeat of the diff. Why does this behaviour need its own change here?
- **INPUT → THIS FILE → RESULT:** a 3-box flow with arrows. INPUT = what feeds in. THIS FILE = what this file does in this PR — name the function or value added. RESULT = what is true afterwards, and who checks it, from where (callers with `file:line`).
- **KEEP THIS DISTINCTION box:** the one misunderstanding to prevent. E.g. "Declaring this client gate does not enable the backend gate or deploy server code."
- **Footer:** `Source: <short sha> | High-level explanation, not proof of backend deployment`. The sha is the PR head commit.
- If a file has no meaningful distinction to warn about, say what it does *not* change instead of dropping the box.

## Worked example — one filled-in file slide (real PR)

This is a complete slide from the sample deck, built from the real diff of **ahmedkhaleel2004/gitdiagram#205 — "Tighten the mobile repo card layout"** (merged, 3 files, +38 −36, head commit `c0d9fe39338572c5991fb1ae133170719a65fd95`). Copy its shape, not its words.

```
PR 205 / 01 of 03
main-card.tsx
src/components/main-card.tsx

1. Mobile action row

┌─ WHY this file changes ─────────────────────────────┐
│ The repo-page card stacked each secondary action at │
│ a different width, and put the zoom label and the   │
│ zoom switch on separate rows — so on a phone the    │
│ card was tall, uneven, and hard to scan.            │
└─────────────────────────────────────────────────────┘

┌─ INPUT ────────┐    ┌─ THIS FILE ────────┐    ┌─ RESULT ────────┐
│ A user opens   │ →  │ MainCard lays out  │ →  │ At 390px the    │
│ a repo page on │    │ the action row     │    │ closed card is  │
│ a phone; the   │    │ (changed lines     │    │ 278px tall      │
│ diagram is     │    │ 104–175):          │    │ (was 376px);    │
│ generated and  │    │ Regenerate +       │    │ expanded export │
│ MainCard       │    │ Export become an   │    │ card 416px      │
│ renders the    │    │ equal two-column   │    │ (was 516px).    │
│ repo card.     │    │ grid on mobile,    │    │ Controls still  │
│                │    │ each a full-width  │    │ fit at 320px    │
│                │    │ 44px button with   │    │ with no         │
│                │    │ a short label      │    │ overflow.       │
│                │    │ ("Regenerate" /    │    │ Checked by      │
│                │    │ "Export"); the     │    │ ExportDropdown  │
│                │    │ zoom label +       │    │ and CopyButton, │
│                │    │ switch become one  │    │ which render    │
│                │    │ aligned row.       │    │ inside this     │
│                │    │ +32 −30            │    │ card.           │
└────────────────┘    └────────────────────┘    └─────────────────┘

┌─ KEEP THIS DISTINCTION ─────────────────────────────┐
│ This only rearranges the mobile layout. It does not │
│ change what Regenerate or Export do, and the desktop│
│ layout and labels are unchanged.                    │
└─────────────────────────────────────────────────────┘

Source: c0d9fe393385 | High-level explanation, not proof of backend deployment
```

The same PR's other two slides follow the same shape: `copy-button.tsx` (02 of 03, +1 −1 — the copy button becomes a full-width 44px touch target on mobile) and `export-dropdown.tsx` (03 of 03, +5 −5 — tighter spacing and smaller metadata text on mobile). The full 7-slide sample deck (cover + phone app-demo + HLD + these three file slides + gates & tests) is the reference output for this skill; its gates slide is honest about provenance — no test files are in the diff, and the 282 tests are reported in the PR validation, not re-run.

## Visual style — Classic Sketchbook (light)

The locked theme. Light, warm, hand-drawn — never dark, never saturated.

- **Background:** warm cream paper `#fffef9`; slides white.
- **Boxes (pale pastels only):**
  - WHY — pale yellow `#fff8d6`
  - INPUT / RESULT — pale blue `#eaf2fe`
  - THIS FILE — pale green `#e8f5eb`
  - KEEP THIS DISTINCTION — pale lavender `#f2eeff`
- **Borders:** thin slate `#3d4a5f`, 1.6–1.8px, slightly wobbly hand-drawn radii; soft shadows only (`rgba` 0.08–0.14). No heavy black borders, no hard shadows.
- **Delta colours (use everywhere, consistently):** green = new, amber = changed, red = removed. Always add a text badge (`+ New` / `Δ Changed` / `− Removed`) so colour is never the only cue.
- **Fonts:** hand-drawn for diagram labels and titles — Excalifont (Excalidraw `fontFamily: 5`, current default) or Virgil (`fontFamily: 1`); clean body text in Inter; code / file paths / line refs in Cascadia or IBM Plex Mono. Diagram text ≥14px, body ≥16px, titles ≥20px. No emoji inside diagrams.
- **Diagrams:** generate Excalidraw JSON (`.excalidraw`) as the canonical artefact — plain JSON array of elements, fixed `seed` for reproducible wobble, `roughness: 1` for hand-drawn. Render with `@excalidraw/excalidraw` (`convertToExcalidrawElements`, `exportToSvg` / `exportToBlob`) or headless via Kroki (`kroki.io/excalidraw/svg`, SVG only). Ship the `.excalidraw` file alongside the deck so it opens in excalidraw.com.
- **Light theme only.** Do not add a dark mode and do not follow the OS dark preference.

## Output formats

- **Primary: PowerPoint deck.** This is the deliverable reviewers receive, forward, and present. Build it natively from the same structured data with PptxGenJS — one slide per deck section, diagrams as full-slide PNGs. PptxGenJS is a drawing API, not an HTML converter: it places every shape in inches, with no HTML/CSS layout and no auto-reflow. Accept that it looks plainer than the HTML twin; do not promise HTML fidelity in PPTX. Marp/Slidev `--pptx` exports slides as flat images (not editable text) — label which kind of PPTX it is.
- **HTML twin: single-file HTML deck.** The same slides with the interactivity PowerPoint cannot keep (Before/After toggle, click a node → the real diff hunk, search, keyboard nav), working offline with fonts fallen back, version-controlled as text. Inline CSS/JS, embedded SVG diagrams, Google-Fonts link with a system-font fallback.
- **Also emit:** the `.excalidraw` JSON, diagrams as standalone SVG/PNG, and a `summary.md` (the PR-comment text).
- **PDF:** free via the browser's print-to-PDF with a print stylesheet; do not build a separate PDF pipeline.

## Verification checklist (run before calling a deck done)

- [ ] Every file in the PR appears on a slide, or is listed as covered-with-reason (`lockfile`, `tests`, `formatting`). Anything not covered is flagged *changed, not explained*.
- [ ] Every function named on a slide exists in the diff at the cited `file:line`.
- [ ] Every caller edge is a parsed edge, or flagged `inferred — verify`.
- [ ] Every gate finding has tool + rule + `file:line`. No gate fires on untouched code. Any gate that did not run is labelled `not run — <tool> missing`, never "pass".
- [ ] Test presence is never presented as a coverage %.
- [ ] Numbers (files, +/−, heights, timings) match the PR body or the API — re-check them, don't trust memory.
- [ ] Mocks and screenshots are labelled as such.
- [ ] Deck opens, all slides navigate, Before/After toggle works, no horizontal overflow on mobile.

## Example

Sample deck built from a real PR: **ahmedkhaleel2004/gitdiagram#205 — "Tighten the mobile repo card layout"** (merged, 3 files, +38 −36). Cover + interactive phone app-demo (closed card 376px → 278px, expanded 516px → 416px at 390px) + HLD + one slide per file (`main-card.tsx` 01/03 core, `copy-button.tsx` 02/03, `export-dropdown.tsx` 03/03) + gates & tests (no test files in the diff; 282 tests reported in the PR validation, not re-run). Head commit `c0d9fe39338572c5991fb1ae133170719a65fd95`. The filled-in `main-card.tsx` slide above is taken from that deck.

## Out of scope

Whole-repo HLDs, hosted apps with auth/billing, auto-fix, and posting review comments as a bot reviewer (do not compete with AI reviewers — link to them). The deck explains; the reviewer decides.
