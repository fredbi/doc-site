# Plan — Repository status dashboard (go-openapi & go-swagger)

> **Status:** draft. Fork-only branch, **never merged upstream** — working design document.
> **Authored:** 2026-05-30 (drafted with Claude).

## 1. Goal

A single, always-current **dashboard** page on the doc site showing every go-openapi + go-swagger repo at a
glance: status, latest release, backlog, activity churn, and quality badges. Organized as **tabs** — the same
repo list seen from different angles — and refreshed **daily with no daily commits**.

## 2. Decisions (resolved 2026-05-30)

- **Engine: a scheduled GitHub Actions cron** (not a Claude routine). This is pure mechanical aggregation
  (API → table); no LLM judgment is involved, and it avoids the routine git-identity constraints.
- **Data store / render: bake at deploy.** The daily workflow collects via GraphQL, writes a Hugo data file,
  builds, and deploys to Pages. **No repo commits, no gist, no client JS.** The data lives only in the
  regenerated static site.
- **Metrics: all four groups** — Release & status, Backlog, Churn, Quality badges.
- **Layout: a tabbed page** (Status / Activity / Quality), same repo list per tab, different columns.

## 3. Architecture

```
   (daily cron / dispatch / content push)          GitHub GraphQL API
        │                                           (gh api graphql, bot token)
        ▼                                                   ▲
  ┌───────────────────────────────┐   one batched query     │
  │  JOB: collect-dashboard       │────────────────────────-┘
  │  perms: contents:read         │   (+ security-events:read ONLY
  │        [+ security-events?]   │      if the vuln metric is on — D-h)
  │  → dashboard.json (artifact)  │
  └───────────────┬───────────────┘
                  │ upload-artifact → download into hugo/data/dashboard.json
                  ▼
  ┌───────────────────────────────┐   reads site.Data.dashboard
  │  JOB: build-doc (existing)    │   renders tabbed tables + badges
  └───────────────┬───────────────┘
                  ▼
  ┌───────────────────────────────┐   perms: pages:write, id-token:write (unchanged)
  │  JOB: deploy-doc (existing)   │   dashboard fresh once/day, zero commits
  └───────────────────────────────┘
```

## 4. Components

### 4.1 Collection workflow

- **Extend the existing `update-doc.yml`** with a daily `schedule:` + `workflow_dispatch:`, and add data
  collection as a **distinct job** (`collect-dashboard`) that the existing `build-doc` job `needs:` — *not*
  an inline build step. Integrating in `update-doc.yml` reuses build/deploy and refreshes the dashboard on
  every content deploy; keeping it a *separate job* isolates its permissions and its failure domain. **[D-a]**
- **Handoff:** `collect-dashboard` writes `dashboard.json` and `actions/upload-artifact`s it; `build-doc`
  `download-artifact`s it into `hack/doc-site/hugo/data/dashboard.json` before the Hugo build.
- **Per-job permissions (least privilege):**
  - `collect-dashboard`: `contents: read` + the bot read token (D-d). **Adds `security-events: read` ONLY if
    the open-vulnerability metric is enabled (D-h)** — the single thing that could need elevated read scope,
    and it stays contained to this one job.
  - `build-doc` / `deploy-doc`: **unchanged** — `deploy-doc` already holds `pages: write` + `id-token: write`;
    the publish side gains nothing.
- **Make the collection job non-fatal** (`continue-on-error`); `build-doc` falls back to an empty dataset if
  the artifact is missing, so a transient GraphQL hiccup degrades the page to "data unavailable" rather than
  failing the site deploy.
- **cron:** daily, e.g. `0 6 * * *` UTC. **[D-e]** Shares the existing `concurrency: pages` group.

### 4.2 The generator (`hack/doc-site/dashboard/`)

- **v1: a shell script using `gh api graphql` + `jq`** (matches "gh graphql"); revisit a Go program (hack/
  already has Go) if transforms grow. **[D-b]**
- **One batched GraphQL query** (org `repositories` connection, or repo aliases) fetches per repo:
  `defaultBranchRef`, latest `releases(first:1)` (tag + `publishedAt`), `issues(states:OPEN){totalCount}`,
  `pullRequests(states:OPEN){totalCount}`, commit `history(since:)`​`{totalCount}` for the 30-day window, and
  merged-PR count (via `search` or filtered `pullRequests`).
- **Repo list:** discover dynamically from both orgs, filtering out archived repos and forks, with an optional
  manual exclude list — so new repos appear automatically. **[D-c]**
- Writes `hack/doc-site/hugo/data/dashboard.json` (ephemeral in CI; gitignored).

### 4.3 Data schema (`dashboard.json`)

```json
{
  "generatedAt": "2026-05-30T06:00:00Z",
  "repos": [
    {
      "org": "go-openapi", "name": "runtime", "url": "https://github.com/go-openapi/runtime",
      "archived": false, "defaultBranch": "master",
      "release": { "tag": "v0.32.2", "publishedAt": "2026-05-27", "ageDays": 3 },
      "openPRs": 0, "openIssues": 1, "commits30d": 87, "prsMerged30d": 23,
      "badges": { "ci": "…", "coverage": "…", "goreport": "…", "license": "…", "release": "…" }
    }
  ]
}
```

One repos array with all fields; each tab renders a different column subset.

### 4.4 The page (committed)

- `docs/doc-site/dashboard/_index.md` + a custom layout/partial that reads `site.Data.dashboard` and renders
  the **Relearn `tabs` shortcode** with three data-driven tables. **[D-g]**
  - **Status:** default branch · latest release (tag + age) · active/archived.
  - **Activity:** open PRs · open issues · commits (30d) · PRs merged (30d).
  - **Quality:** CI/build · coverage · Go Report Card · license · latest-release badges
    *(optional: an open-vulnerability count — the only metric needing elevated read scope; see D-h. Could live
    here or in its own "Security" tab.)*
- **Badges are templated image URLs** per repo (Go Report Card, CodeQL/CI, codecov, license, release shields)
  — computed by the badge services, no API collection needed. The generator can emit the URLs, or the
  template can build them from the repo name. **[D-f: exact columns above are a proposal]**
- Footer line: "Last updated: {generatedAt}".

## 5. Open decisions (have recommendations)

- **D-a** — integrate into `update-doc.yml` vs separate workflow → **integrate** (reuse build/deploy).
- **D-b** — generator language → **`gh api graphql` + `jq`** for v1.
- **D-c** — repo list → **dynamic discovery**, filter archived/forks, optional manual excludes.
- **D-d** — token → least-privilege **`go-openapi-bot` read PAT** (cross-org metadata read), matching the
  "executed by go-openapi-bot" intent; `GITHUB_TOKEN` would also read public data but the bot PAT is cleaner
  for cross-org reads and identity. (Publish permissions are unchanged — see §4.1.)
- **D-e** — cron time (daily; exact hour TBD).
- **D-f** — exact columns per tab (proposal in §4.4).
- **D-g** — confirm the Relearn theme renders data-driven tables inside `tabs` via a custom partial (it does).
- **D-h** — **include an open-vulnerability count? (undecided).** The only metric needing elevated read scope:
  Dependabot/code-scanning alerts require `security-events: read` across all repos (a bot PAT with
  `security_events` / fine-grained "Dependabot alerts: read", or a GitHub App). If included, the scope stays
  confined to the `collect-dashboard` job; everything else needs only public-metadata read.

## 6. Setup checklist

1. Add the generator (`hack/doc-site/dashboard/…`) + the dashboard page/layout (one normal PR).
2. Add the daily `schedule` + `workflow_dispatch` + the non-fatal generate step to `update-doc.yml`.
3. Add the bot read-token secret (unless `GITHUB_TOKEN` suffices).
4. **Dry run** via `workflow_dispatch`; verify the rendered tabs, badges, and data freshness, then enable cron.

## 7. Risks & constraints

- **Daily whole-site redeploy** — acceptable (Pages build is cheap; `concurrency: pages` already serializes).
- **GraphQL points** — one batched query/day is well within 5,000/hr.
- **Coupling** — generating inside `update-doc.yml` couples content deploys to the API; the non-fatal step
  (§4.1) removes that risk.
- **Permission isolation** — collection is a distinct job, so any elevated read scope (e.g. the optional vuln
  count) is confined to it; `build-doc`/`deploy-doc` keep their existing minimal permissions.
- **Dynamic discovery** may surface unwanted repos → filter + manual exclude list.
- **External badge services** (shields, goreportcard, codecov) — cosmetic if temporarily down.
- **Data is ephemeral** (not versioned) — by design; the generator script is the source of truth.

## 8. Out of scope (for now)

- **Historical trends / time-series** — would need a persistent store (a gist or a `data` branch); revisit if
  we want sparklines or week-over-week deltas.
- Per-repo drill-down pages.
- **Cross-repo event-driven triggers** — explicitly rejected as too complex; the daily cron is enough.
