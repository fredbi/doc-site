# Plan — Automated monthly reports (go-openapi & go-swagger)

> **Status:** draft. This plan lives on a fork branch and is **never merged upstream** — it is a working
> design document, committed for the record only.
> **Authored:** 2026-05-30 (drafted with Claude).

## 1. Goal

Produce **timely, short, fact-led monthly reports** of activity across all go-openapi and go-swagger
repositories, published under `docs/doc-site/blog/monthly/`, **without a high-touch monthly writing session**.

This complements — does not replace — the existing **quarterly** report, which stays human-driven and mixes
facts with strategic narrative. The monthly is the low-effort, recent-progress counterpart:

| | Quarterly (existing) | Monthly (this plan) |
|---|---|---|
| Cadence | Once a quarter | Once a month (the 5th, covering the previous calendar month) |
| Tone | Facts **+** grand-strategy narrative | Mostly facts, light framing, no grand strategy |
| Effort | Intense human + Claude session | Autonomous draft → human review/merge |
| Scope | All repos (go-openapi + go-swagger) | Same scope, shorter window |
| Trigger | Manual | Scheduled **routine** |

## 2. Architecture

```
  ┌─────────────────────────┐     (5th of month, cron)
  │  Claude routine (cloud)  │◀──────────────────────────────┐
  │  - clones doc-site       │                                │
  │  - invokes monthly skill │   gh/git over all repos        │  /schedule
  │  - writes monthly/*.md   │──────────────▶ GitHub API      │  (user-owned)
  │  - opens PR (claude/*)   │                                │
  └───────────┬─────────────┘                                 │
              │ PR to doc-site                                 │
              ▼                                                │
  ┌─────────────────────────┐  human reviews / edits / merges │
  │  Pull request            │────────────────────────────────┘
  └───────────┬─────────────┘
              │ merge to master → "Update documentation" workflow builds + deploys Pages
              ▼
  ┌─────────────────────────┐
  │ announce-monthly.yml     │  (near-identical to announce-quarterly.yml)
  │ posts to #monthly-news   │
  └─────────────────────────┘
```

Three new pieces, each modeled on something we already have:

1. **`monthly-newsletter` skill** — adapted from `quarterly-newsletter` for tone & concision.
2. **The routine** — a scheduled cloud agent that runs the skill and opens the PR.
3. **`announce-monthly.yml`** — the Discord announcer, cloned from `announce-quarterly.yml`.

## 3. How routines work (the new capability)

Reference: <https://code.claude.com/docs/en/routines.md>, <https://code.claude.com/docs/en/claude-code-on-the-web.md>.

- **Runtime:** fresh Anthropic-managed cloud VM (Ubuntu 24.04) per run; clones the routine's repo (doc-site)
  from its **default branch** and auto-loads that repo's `.claude/` config (skills, rules, agents, settings).
- **Schedule:** cron, **minimum interval 1 hour**. `0 9 5 * *` ≈ 5th @ 09:00 (entered local, stored UTC; small
  stagger). Covers the previous calendar month.
- **Tooling:** git, `jq`, `yq`, ripgrep preinstalled. **`gh` is NOT** — install via the environment setup
  script (`apt-get update && apt-get install -y gh`) and provide a `GH_TOKEN` env var. Network defaults to
  "Trusted" (GitHub + package registries allowed) — enough to clone public repos and use the GitHub API.
- **Autonomous PR:** routines can push a branch and `gh pr create` **with no approval gate**. Default push is
  restricted to `claude/*` branches (can be lifted per repo). PR is attributed to the connected GitHub
  identity (GitHub App install, or a token synced with `/web-setup`).
- **Monitoring:** runs appear at `claude.ai/code/routines` as full sessions (transcript + diff). **No built-in
  notifications** — the opened PR is the signal. "Green" means it started, not that it succeeded.
- **Ownership / limits:** the routine is created and owned by a human via `/schedule` (web/CLI) — Claude cannot
  provision it. Daily run cap; counts against subscription usage. **Research preview — limits/API may change.**

## 4. Component design

### 4.1 `monthly-newsletter` skill (`.claude/skills/monthly-newsletter.md`)

Fork of `quarterly-newsletter` with these deltas:

- **Window:** previous calendar month (`--since` = first day of last month, `--until` = first day of this month).
- **Tone:** factual and concise. Drop the strategy narrative, the impact-assessment essay, and the long
  per-repo prose. Keep: a one-paragraph intro, a compact themes list, a short repo-highlights table, the
  contributor thanks, and the front-matter `description` + `discord_description` fields.
- **Length target:** roughly a third of a quarterly — skimmable in a minute.
- **Data sourcing for the cloud env:** prefer the **GitHub API** (`repos/<owner>/<repo>/commits`,
  `.../releases`, `.../pulls`, `.../issues`) over local clones, since the routine starts with only doc-site
  checked out. Shallow-clone (`--depth`/`--shallow-since`) only if a repo needs file-level inspection.
  **Reads must be authenticated** — unauthenticated GitHub API is 60 req/hour, far too low for sweeping ~18
  repos; authenticated is 5,000 req/hour. Use the GitHub App's built-in GitHub tools (no stored token) or a
  `GH_TOKEN` for the `gh` CLI (see §4.3 / decision D3).
- **Front matter:** same dual-field convention as quarterly — plain `description` for Hugo cards, rich
  `discord_description` for the announcement.
- **No human-in-the-loop assumptions:** the skill must run unattended end-to-end (no questions), unlike the
  quarterly which interviews the maintainer. Any "strategy" is omitted rather than asked for.

### 4.2 Monthly report file & naming

- Path: `docs/doc-site/blog/monthly/<YYYY>-<MM>.md` (e.g. `2026-05.md`).
- Hugo slug → `2026-05`; published URL → `…/blog/monthly/2026-05/index.html`.
- `weight`: descending by date so newest sorts first — proposal: `weight = 999999 - (YYYY*100 + MM)`
  (or simply maintain a small decrement like quarterly). **[decision]**

### 4.3 The routine

- **Repo:** `go-openapi/doc-site` (so it gets the skill + writes to the blog + targets the right PR).
- **Schedule:** `0 9 5 * *` (adjust local time).
- **Environment:**
  - Network: Trusted.
  - Auth: the **Claude GitHub App** (already installed) for clone/push/PR and authenticated reads via its
    built-in GitHub tools. Add a `GH_TOKEN` env var **only** if we keep the skill's `gh` CLI calls (it needs
    one); otherwise adapt the skill to the built-in GitHub tools and store no token (preferred).
  - Setup script: install `gh` (only if the CLI path is used).
  - Branch pushes: keep the default `claude/*` restriction (PR from `claude/monthly-YYYY-MM`).
- **Prompt (outline):**
  1. Invoke the `monthly-newsletter` skill for the previous calendar month.
  2. Write `docs/doc-site/blog/monthly/<YYYY>-<MM>.md`.
  3. Commit on `claude/monthly-<YYYY>-<MM>`, push, open a PR to `master` titled
     "doc(blog): monthly report — <Month YYYY>".
  4. Do **not** post to Discord (that happens on merge, via the workflow).

### 4.4 `announce-monthly.yml`

Copy of `announce-quarterly.yml` with:

- Path filter (in-job `git diff --diff-filter=A`): `docs/doc-site/blog/monthly/[^/]+\.md`.
- URL: `…/blog/monthly/<slug>/index.html`.
- Channel: **#monthly-news** via a new secret **`DISCORD_MONTHLY_WEBHOOK_URL`**.
- Same chaining off the "Update documentation" workflow (announce only after Pages deploy succeeds).
- Same `discord_description` extraction.

## 5. Decisions

**Resolved (2026-05-30):**

- **D1 — DCO / human-author rule: NOT a violation.** The agent (Claude, in the cloud) produces the content,
  but the work is **sponsored and validated by the maintainer**, exactly like the commits we already make
  together: `author: fredbi`, `Co-Authored-By: Claude` (read: human-sponsor = fred, content-producer =
  Claude). Analogy: a contractor writing code for a client — the client owns and authors it even though
  Claude's "fingers play the music." Mechanics: routine commits on `claude/monthly-*` are authored by the
  connected GitHub identity (fredbi) with the Claude co-author trailer; the maintainer **squash-merges** so
  master carries fredbi's authorship + DCO sign-off. No rule change needed.
- **D3 — Auth: use the installed Claude GitHub App.** Reasoning: auth is needed both to **open the PR** (push
  + create) and, importantly, to make **authenticated reads** across ~18 repos — unauthenticated GitHub API
  is capped at 60 req/hour, authenticated at 5,000. The App provides push/PR + built-in authenticated GitHub
  read tools with no stored token. A `GH_TOKEN` is added only if we retain the skill's `gh` CLI calls.
- **D4 — Quiet months / quarter overlap: always publish, note the overlap.** Publish every month regardless
  of volume. In quarter-end months, keep the monthly factual and let the quarterly carry the strategy; a one
  line note can point readers to the quarterly.

**Still open (low stakes, have recommendations):**

- **D2 — PR target.** Recommend upstream `go-openapi/doc-site:master`, maintainer merges.
- **D5 — `weight` scheme for monthly** (see §4.2): descending by `YYYYMM`.
- **D6 — Failure handling.** No auto-retry; maintainer clicks "Run now" if the 5th's run fails. Acceptable
  for a monthly cadence.

## 6. Setup checklist (human-owned steps)

1. Land the `monthly-newsletter` skill + `announce-monthly.yml` on `master` (normal PR).
2. Create the `#monthly-news` Discord channel + webhook; add `DISCORD_MONTHLY_WEBHOOK_URL` repo secret.
3. Confirm the Claude GitHub App is installed on `doc-site` (and granted org read for cross-repo data). Add a
   `GH_TOKEN` env var only if the skill keeps its `gh` CLI calls.
4. Create the routine via `/schedule` (or claude.ai/code/routines) against `go-openapi/doc-site`, with the
   schedule, env (network=Trusted, `gh` install, `GH_TOKEN`), and prompt from §4.3.
5. **Dry run:** use "Run now" once; verify the PR, the rendered page, and (after merge) the Discord post.

## 7. Risks & constraints

- **Research preview** — routine behavior/limits may change; keep the design tolerant.
- **Quality drift** — an unattended factual report can misread commit intent; the human review gate is the
  backstop. Keep the skill conservative (report what's verifiable, avoid editorializing).
- **Cross-repo data volume** — favor the GitHub API over cloning 18 repos; cap and `log` anything truncated.
- **Secret handling** — routine env vars are team-visible; use a least-privilege, rotatable token.
- **Cost** — counts against subscription usage + daily routine cap; one run/month is negligible.

## 8. Out of scope (for now)

- Weekly reports (the `#weekly-news` channel exists but is not in this plan).
- A shared/reusable routine across orgs — this is doc-site-specific.
- Auto-merge of the monthly PR — explicitly human-gated.
