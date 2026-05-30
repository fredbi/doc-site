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
  ┌──────────────────────────────┐     (5th of month, cron)
  │  Claude routine (cloud)       │◀───────────────────────────┐
  │  - clones doc-site (App auth) │                             │
  │  - reads all repos via gh     │   gh api (bot GH_TOKEN)     │  /schedule
  │    (bot GH_TOKEN, 5k req/hr)  │──────────────▶ GitHub API   │  (user-owned)
  │  - builds monthly/<YYYY-MM>.md│                             │
  │  - commits file via Contents  │   bot-authored + Verified   │
  │    API to BOT fork (bot tok)  │──────────────▶ bot fork     │
  │  - gh pr create → upstream    │                             │
  └───────────┬──────────────────┘                             │
              │ PR (bot fork → go-openapi/doc-site)             │
              ▼                                                 │
  ┌─────────────────────────┐  human reviews / edits / squash- │
  │  Pull request           │  merges (final master commit     │
  └───────────┬─────────────┘  authored + signed at merge) ────┘
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
  restricted to `claude/*` branches (can be lifted per repo).
- **Git-identity caveat (verified against the docs):** `git push` from the sandbox flows through a GitHub
  **proxy** and is attributed to the *connected* identity (the Claude App / `/web-setup` token). A `GH_TOKEN`
  env var is used by the **`gh` CLI only — not by `git push`** — and a token-in-remote-URL is overridden by
  the proxy. In-sandbox GPG signing keys are also unavailable. **Consequence:** to attribute commits to a
  dedicated **bot** identity *and* get GitHub-Verified signing, do **not** `git push` the report — create the
  commit through the **GitHub Contents API** with the bot's token (`gh api --method PUT …/contents/…`).
  API-created commits are authored by the token owner (the bot) and auto-signed by GitHub (Verified). The
  connected App/`/web-setup` auth is still required for the routine to clone and operate. See §4.3.
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
  repos; authenticated is 5,000 req/hour. Use the **bot's `GH_TOKEN`** with the `gh` CLI/API (same identity
  that authors the report commit; see §4.3 / D3).
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

- **Repo:** `go-openapi/doc-site` (so it gets the skill + targets the right upstream PR base).
- **Schedule:** `0 9 5 * *` (adjust local time).
- **Identity model (see D3):** a dedicated **bot user** (a distinct GitHub account that is an "emanation of
  fredbi") owns a **public fork** of doc-site. The routine writes the report and commits it **to the bot's
  fork via the GitHub Contents API using the bot's `GH_TOKEN`** — *not* `git push` (which the proxy would
  attribute to the connected App identity). API commits are bot-authored and GitHub-**Verified**. It then
  opens a cross-fork PR with `gh`. The bot needs **no org membership**.
- **Environment:**
  - Network: Trusted.
  - Connected auth: the **Claude GitHub App** (already installed) — required so the routine can clone/operate.
  - `GH_TOKEN`: a **fine-grained PAT owned by the bot** (contents + pull-requests write on the bot's fork;
    pull-requests write on upstream to open the PR; read on the org for cross-repo data). Drives all `gh`
    reads, the Contents-API commit, and `gh pr create`.
  - Setup script: install `gh` (`apt-get update && apt-get install -y gh`).
- **Prompt (outline):**
  1. Invoke the `monthly-newsletter` skill for the previous calendar month (reads via the bot `GH_TOKEN`).
  2. Build `docs/doc-site/blog/monthly/<YYYY>-<MM>.md` locally.
  3. Create branch `claude/monthly-<YYYY>-<MM>` on the bot's fork and commit the file via
     `gh api --method PUT repos/<bot>/doc-site/contents/…` (bot-authored, Verified).
  4. `gh pr create --repo go-openapi/doc-site --head <bot>:claude/monthly-<YYYY>-<MM> --base master`
     titled "doc(blog): monthly report — <Month YYYY>".
  5. Do **not** post to Discord (that happens on merge, via the workflow).

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
  but the work is **sponsored and validated by the maintainer**. Analogy fredbi uses: a contractor coding for
  a client — the client owns and authors it even though Claude's "fingers play the music." The dedicated bot
  identity (D3) is the on-the-record stand-in for that sponsored authorship (its profile states it is an
  emanation of fredbi), exactly like `go-openapi-bot` already authors the automated `contributors.md` updates.
  The maintainer **squash-merges** the PR, so the final master commit is authored + signed at merge time.
- **D3 — Identity: a dedicated bot user + public fork (NOT the personal account or the Claude App identity).**
  Verified mechanics that drive this:
  - `git push` in a routine flows through a GitHub proxy and is attributed to the *connected* identity (Claude
    App / `/web-setup` token); a `GH_TOKEN` is used by the `gh` CLI **only**, and token-in-URL is overridden.
    In-sandbox signing keys are unavailable.
  - **Therefore** the report commit is created via the **GitHub Contents API** with the **bot's `GH_TOKEN`**
    (not `git push`): the commit is **bot-authored and GitHub-Verified**, on a `claude/*` branch of the bot's
    **public fork**, and the PR is opened cross-fork with `gh`. This needs **no org membership** and recovers
    signed/verified commits without keys in the sandbox.
  - The Claude GitHub App stays installed (required for the routine to clone/operate); the separate
    identity-switching App used for `go-openapi-bot` in CI workflows is **not needed** here (we're only
    opening a PR, and the bot PAT carries the identity).
  - Authenticated reads across ~18 repos also use the bot `GH_TOKEN` (60 → 5,000 req/hour).
- **D4 — Quiet months / quarter overlap: always publish, note the overlap.** Publish every month regardless
  of volume. In quarter-end months, keep the monthly factual and let the quarterly carry the strategy; a one
  line note can point readers to the quarterly.
- **D7 — Squash-merge author: the bot.** The bot authors the master commit (its signature names fredbi),
  consistent with how `go-openapi-bot` already authors automated commits. There is no goal of maximizing
  fredbi's personal commit count, so bot authorship is fine.

**Still open (low stakes, have recommendations):**

- **D2 — PR target.** Upstream `go-openapi/doc-site:master` (cross-fork from the bot), maintainer merges.
- **D5 — `weight` scheme for monthly** (see §4.2): descending by `YYYYMM`.
- **D6 — Failure handling.** No auto-retry; maintainer clicks "Run now" if the 5th's run fails. Acceptable
  for a monthly cadence.
- **D8 — Verify in dry-run.** The Contents-API-as-bot + cross-fork-PR flow rests on documented behavior plus
  a couple of flagged uncertainties (proxy not touching API commits; cross-fork PR from a routine). Confirm
  on the first "Run now" before trusting the schedule.

## 6. Setup checklist (human-owned steps)

1. Land the `monthly-newsletter` skill + `announce-monthly.yml` on `master` (normal PR).
2. Create the `#monthly-news` Discord channel + webhook; add `DISCORD_MONTHLY_WEBHOOK_URL` repo secret.
3. **Create/confirm the bot identity:** a dedicated GitHub user (profile: "emanation of fredbi"), give it a
   **public fork** of `doc-site`, and mint a **fine-grained PAT** (contents + PRs write on the fork, PRs write
   on upstream, org read). Confirm the Claude GitHub App is installed on `doc-site`.
4. Create the routine via `/schedule` (or claude.ai/code/routines) against `go-openapi/doc-site`, with the
   schedule, env (network=Trusted, `gh` install, bot `GH_TOKEN`), and prompt from §4.3.
5. **Dry run (covers D8):** use "Run now" once; verify the commit is bot-authored + Verified on the fork, the
   cross-fork PR opened, the rendered page, and (after merge) the Discord post.

## 7. Risks & constraints

- **Research preview** — routine behavior/limits may change; keep the design tolerant.
- **Quality drift** — an unattended factual report can misread commit intent; the human review gate is the
  backstop. Keep the skill conservative (report what's verifiable, avoid editorializing).
- **Cross-repo data volume** — favor the GitHub API over cloning 18 repos; cap and `log` anything truncated.
- **Secret handling** — routine env vars are team-visible; use a least-privilege, rotatable token.
- **Cost** — counts against subscription usage + daily routine cap; one run/month is negligible.

## 8. Out of scope

- **Weekly reports — dropped, permanently.** The Discord server is already fed noisily by PR / issue / release
  events, so a weekly digest would be ignored by most members. Monthly + quarterly is the right cadence.
- A shared/reusable routine across orgs — this is doc-site-specific.
- Auto-merge of the monthly PR — explicitly human-gated.
