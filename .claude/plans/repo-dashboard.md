# Plan — Repository status dashboard (go-openapi & go-swagger)

> **Status:** in progress. Fork-only branch, **never merged upstream** — working design document.
> **Authored:** 2026-05-30 (drafted with Claude).
> **Updated:** 2026-05-31 — implementation kickoff. **Rendering POC complete** (see §9): the data-driven
> tabs render end-to-end against a real Hugo build, resolving the one open feasibility risk (D-g). Scope
> for v1 narrowed to **lean core** (Status / Activity / Quality) with the heavier metrics deferred. The
> generator and the workflow wiring are the remaining work.

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
- **Metrics: all four groups** — Release & status, Backlog, Churn, Quality badges — but delivered in
  **phases** (see 2026-05-31 below), not all at once.
- **Layout: a tabbed page** (Status / Activity / Quality), same repo list per tab, different columns.

### Resolved 2026-05-31 (implementation kickoff)

- **v1 = lean core.** Ship **three tabs — Status / Activity / Quality** — populated only from
  *cheap-GraphQL* and *badge-URL* columns. The richer sample (`sample-dashboard.md`) tabs **Go**, **Github**,
  **Security**, **Shipping** and the heavier metrics are explicitly **phase 2+** (see §4.5). Grow tab-by-tab
  once the pipeline is proven.
- **Defer the expensive metrics.** *Commits-excluding-bots, commits MTD/YTD, total contributors,
  protected-master, check-status-enforced, releases-YTD* are **out of v1**: they need REST + pagination or
  author-filtering that one batched GraphQL query can't cheaply provide. Render as `N/A` / omit for now.
- **Rendering mechanism: a custom `dashboard` shortcode + per-tab partials**, feeding the theme's own
  `shortcodes/tabs.html` partial. Confirmed working against a real Hugo build — **D-g resolved** (§9).
- **Badge URLs are templated in the partials** from `org/name` (not stored in the data file). The generator
  collects *facts*; the templates build *presentation*.

### Resolved 2026-05-31 (round 2 — feedback on live data)

After running the generator against real org data, scope was expanded from the lean core:

- **Activity tab → full metric set, incl. the excl-bots split.** Commits MTD / YTD shown both raw and
  excluding bot authors, plus Releases YTD and Total contributors. Computed from a **REST commit pull since
  the start of the year** (MTD ⊂ YTD, so one fetch yields all four counts); bot = `author.type=="Bot"` or
  login/name ending `[bot]`. **Contributors** via the REST `contributors` Link-header `rel="last"` trick.
  This overrides the earlier "defer expensive metrics" call.
- **Open issues are split** into actionable vs deferred (`v2` / `future/maybe` labels), via a GraphQL
  `search` OR-query per repo.
- **Per-repo workflow filenames** (CI / cut-release / CodeQL) live in a generator **overrides table** with
  conventional defaults — `ci-workflows`→`local-*`, `gh-actions`→`test.yml`, `doc-site`→`update-doc.yml` CI &
  `N/A` release. Emitted into the data as a `workflows` map; `null` ⇒ the template renders N/A. This resolves
  the §4.2 "assumed filename" wrinkle.
- **Lint status** has no native per-job badge, so the generator reads the `lint` job's conclusion from the
  latest CI run (Actions runs+jobs API) and the template renders a **self-made shields static badge**
  (`lint-passing-brightgreen` / `lint-failing-red` / `lint-unknown-lightgrey`).
- **Status tab** gains a **Last updated** column (`defaultBranchRef.target.committedDate`).
- **License moved to a new Github tab** (administrative metadata: description, topics, default branch,
  license, stars, forks) — cheap, all from discovery. Protected-branch / required-checks remain phase 2.

### Resolved 2026-05-31 (round 3 — more live-data feedback)

- **External links open in a new tab.** All dashboard anchors carry `target="_blank" rel="noopener"`,
  matching the theme's `externalLinkTarget: _blank` (which only applies to markdown links, not our raw HTML).
- **Workflow overrides extended:** `ci-workflows` CI is `local-go-test.yml` (corrected); `go-swagger/go-swagger`
  CI is `test.yml`; `homebrew-go-swagger`, `scan-repo-boundary`, `go-swagger.github.io` have **CI & cut-release
  = N/A** (`null`). Lint status follows the corrected CI filename automatically.
- **Activity totals:** each org section ends with a **Subtotal** row and the table with a grand **Total** row
  (distinct background, "Cut a release" left blank). Most columns sum; **contributors do not** — that cell is
  the count of **distinct** contributors over the scope (union of logins), supplied by the generator as
  `contributorsDistinct` (per org + `all`). The generator now fetches the contributor **login list** per repo
  (paginated, real users only — no anonymous) to compute both the per-repo count and the distinct unions.
- **Archived repos are now collected, not filtered.** They are excluded from the active tables but listed in a
  compact **archived table** on the Github tab (Repo · Tags · Description · Archived on, the real date from
  `Repository.archivedAt`). Enrichment is skipped for them (no API cost).

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
- **One batched GraphQL query** (org `repositories` connection, or repo aliases) fetches per repo, for
  **v1 lean core**: `name`, `url`, `description`, `isArchived`, `isFork`, `defaultBranchRef`,
  `stargazerCount`, `forkCount`, `repositoryTopics`, `licenseInfo`, latest `releases(first:1)`
  (tag + `publishedAt`), `issues(states:OPEN){totalCount}`, `pullRequests(states:OPEN){totalCount}`, total
  commit `history{totalCount}`, and commits-since-release via `history(since: <release publishedAt>)`​`{totalCount}`
  on the default branch.
- **Phase 2 — per-repo enrichment** (added round 2): a loop over the discovered repos fetches, per repo,
  commits-since-release (GraphQL `history(since:)`), commit windows MTD/YTD ± bots (**REST** `commits?since=`
  start-of-year, partitioned in jq), releases-YTD (GraphQL), deferred-issue count (GraphQL `search`),
  contributor logins (**REST**, paginated → per-repo count + per-scope **distinct** union) and the `lint` job
  conclusion (**REST** Actions runs+jobs). Each
  enrichment is **non-fatal** — failures fall back to a neutral default so one flaky repo never aborts the run.
- **Repo list:** discover dynamically from both orgs, filtering out **forks** and the manual exclude list — so
  new repos appear automatically. A **fork include-list** keeps first-party forks that the fork filter would
  otherwise drop (e.g. `testify`, the maintained testify/v2 fork — GitHub flags it `isFork:true`). **Archived
  repos are kept** (enrichment skipped) so the Github tab can list them; the active tables filter them out. **[D-c]**
- **Workflow filenames:** the CI / cut-release / CodeQL badge filenames come from a **generator overrides
  table** (conventional defaults + per-repo exceptions), emitted as each repo's `workflows` map; `null` ⇒
  N/A. Resolves the earlier "assumed filename" wrinkle. *(Still hard-coded in the script, not auto-detected —
  a maintained table, which is fine for a handful of exceptions.)*
- Writes `hack/doc-site/hugo/data/dashboard.json` (ephemeral in CI; gitignored — already added to
  `hack/doc-site/hugo/.gitignore`).

### 4.3 Data schema (`dashboard.json`)

v1 lean-core schema (as consumed by the POC partials). Badge URLs are **not** stored — the templates build
them from `org`/`name`. `orgs` drives the per-org grouping order; `release` is `null` for unreleased repos
(`hasRelease: false`).

```json
{
  "generatedAt": "2026-05-31T06:00:00Z",
  "orgs": ["go-openapi", "go-swagger"],
  "contributorsDistinct": { "go-openapi": 96, "go-swagger": 410, "all": 470 },
  "repos": [
    {
      "org": "go-openapi", "name": "runtime", "url": "https://github.com/go-openapi/runtime",
      "description": "Runtime middleware", "archived": false, "isFork": false,
      "defaultBranch": "master", "topics": ["openapi", "swagger2"], "license": "Apache-2.0",
      "lastCommitAt": "2026-05-30T18:22:00Z",
      "workflows": { "ci": "go-test.yml", "release": "bump-release.yml", "codeql": "codeql.yml" },
      "hasRelease": true,
      "release": { "tag": "v0.32.2", "publishedAt": "2026-05-27T09:14:00Z", "ageDays": 3 },
      "openPRs": 0, "openIssues": 1, "openIssuesDeferred": 0,
      "commitsSinceRelease": 12,
      "commitsMTD": 8, "commitsMTDNoBot": 5, "commitsYTD": 140, "commitsYTDNoBot": 96,
      "releasesYTD": 4, "totalCommits": 1206, "contributors": 37,
      "lint": "success",
      "stars": 1025, "forks": 23
    }
  ]
}
```

`workflows.*` is `null` where not applicable (→ N/A). `openIssues` is the total; the Activity tab shows
`openIssues - openIssuesDeferred` (actionable) and `openIssuesDeferred` (v2/future). `lint` ∈
{`success`, `failure`, `cancelled`, `skipped`, `unknown`}.

One repos array with all fields; each tab renders a different column subset. Phase-2 metrics (§4.5) add
fields to this same object rather than restructuring it.

### 4.4 The page (committed)

**Built and verified in the POC (§9).** `docs/doc-site/dashboard/_index.md` invokes a custom
`{{</* dashboard */>}}` shortcode (`layouts/shortcodes/dashboard.html`). The shortcode reads
`site.Data.dashboard`, builds a tab slice via three per-tab partials, and hands it to the **theme's own**
`partial "shortcodes/tabs.html"` — so the output is identical to hand-authored `tabs`, but data-driven. **[D-g
resolved]** If the data file is missing, it degrades to a "data unavailable" notice instead of breaking.

Files (committed):
`layouts/shortcodes/dashboard.html`, `layouts/partials/dashboard/{status,activity,quality,github}.html`,
`docs/doc-site/dashboard/_index.md` (page + legend).

Columns as implemented (per-org grouped rows; **four tabs**):
- **Status:** Repo · Release (tag) · CI badge · Last updated · Commits since release (`❗` if >20) · Cut-a-release.
- **Activity:** Repo · Open PRs · Open issues · Open issues (v2/future) · Commits since release · Commits MTD ·
  MTD excl. bots · Commits YTD · YTD excl. bots · Releases YTD · Total commits · Total contributors · Cut-a-release.
- **Quality:** Repo · Test Coverage · *Code quality* {CodeFactor · GoReportCard} (spanning 2-row header) ·
  CodeQL · Linter config (`.golangci.yml` link) · Lint status (self-made badge).
- **Github:** Repo · Description · Tags (topics) · License · Default branch · Stars · Forks — plus a compact
  **archived-repos** table below (Repo · Tags · Description · Archived on).

Activity carries per-org **Subtotal** rows and a grand **Total** row. All external links open in a new tab.

- **Badges are templated image URLs** built **in the partials** from `org`/`name` (codecov, CodeFactor,
  GoReportCard, shields license/lint, GitHub Actions) — computed by the badge services or self-rendered, no
  extra collection. **[D-f resolved; phase-2 tabs (Go, Security, Shipping) add more.]**
- Footer line: "Last updated: {generatedAt}" — rendered from `time.AsTime`.

### 4.5 Deferred tabs & metrics (phase 2+)

From the richer `sample-dashboard.md`, **still not** built (round 2 pulled the Activity metrics and a Github
tab forward — see §2):
- **Tabs:** **Go** (go version, godoc, mono-repo, top languages), **Security** (policy, Dependabot, open
  vulnerabilities — *open-vulnerability count delivered 2026-06-02 as two Github-tab columns, see
  `dashboard-security.md` / D-h; a dedicated Security tab is still deferred*), **Shipping** (Docker/package
  repos, go-swagger-specific).
- **Github tab extras:** **protected-master** and **required-checks** status (need the branch-protection API).
- **Fork-aware metrics for `testify`** — ✅ **DONE (2026-06-02, branch `feat/dashboard-sec`).** As a fork it
  is included via the allow-list, but GitHub reported the whole fork network for **Total commits** and **Total
  contributors** (~1483 / 258 vs our own ~227 / 6). The generator now counts only the fork's own commits since
  the fork point via the cross-fork compare API (`parent_default...fork_default`), overriding `totalCommits`
  and the contributor list (so the per-org / overall *distinct* totals de-inflate too). The **commit windows**
  needed no change after all: testify's history holds only shared ancestry (dated before the fork point) plus
  our own commits, so the `since=` date filters already exclude upstream. Needs the discovery query's new
  `parent { nameWithOwner defaultBranchRef }` field.
- **Auto-detected workflow paths** — the overrides table (§4.2) is maintained by hand; auto-detection would
  remove the need to edit the script when a repo diverges. Low priority.

## 5. Open decisions (have recommendations)

- **D-a** — integrate into `update-doc.yml` vs separate workflow → **integrate** (reuse build/deploy).
- **D-b** — generator language → **`gh api graphql` + `jq`** for v1.
- **D-c** — repo list → **dynamic discovery**, filter archived/forks, optional manual excludes.
- **D-d** — token → **RESOLVED**: `collect-dashboard` uses `secrets.DASHBOARD_READ_TOKEN || github.token`. The
  default `github.token` reads all the public cross-org data we need (metadata, commits, issues, search,
  Actions runs of public repos), so v1 needs no secret; set `DASHBOARD_READ_TOKEN` to a least-privilege bot
  PAT later for a dedicated identity. (Publish permissions unchanged — §4.1.)
- **D-e** — cron time → **RESOLVED**: daily at **06:00 UTC** (`0 6 * * *`).
- **D-f** — exact columns per tab → **RESOLVED for v1** (columns implemented in §4.4); phase-2 tabs revisit.
- **D-g** — render data-driven tables inside `tabs` via a custom partial → **RESOLVED**: a custom `dashboard`
  shortcode feeds the theme's `shortcodes/tabs.html` partial; verified against a real Hugo build (§9).
- **D-h** — include an open-vulnerability count? → **RESOLVED (2026-06-02): yes** — see
  `dashboard-security.md`. Two Github-tab columns: **Security alerts** (combined open count across code
  scanning / Dependabot / secret scanning) and **Security reports** (open repository advisories). The elevated
  read is confined to a separate **`SECURITY_TOKEN`** used only by those four calls — a **go-openapi-bot App
  installation token** (installed on both orgs; reads code scanning / Dependabot / secret scanning alerts +
  repository security advisories), not the job's `GITHUB_TOKEN` (which is doc-site-scoped and cannot read other
  repos' security data). A flavor we cannot read is flagged `⚠️`, distinct from a clean `0`.

## 6. Setup checklist

1. ✅ **Dashboard page/layout** — `_index.md` + `dashboard` shortcode + per-tab partials. **Done** (POC, §9).
2. ✅ **Generator** (`hack/doc-site/dashboard/collect-dashboard.sh`) — discovery + enrichment producing the
   §4.3 schema. **Done** and validated against live org data.
3. ✅ **Workflow wiring** — daily `schedule` (06:00 UTC) + `workflow_dispatch` + the non-fatal
   `collect-dashboard` job feeding `build-doc` via a `dashboard-data` artifact. **Done.**
4. ✅ **Token** — none required: `collect-dashboard` uses `secrets.DASHBOARD_READ_TOKEN || github.token`, and
   the default token reads all the public data we need. Set the secret later for a dedicated bot identity.
5. ⏳ **Dry run** via `workflow_dispatch` on the branch; verify the rendered tabs/badges and data freshness in
   the deployed (or PR-preview) site, then let the daily cron take over.

## 7. Risks & constraints

- **Daily whole-site redeploy** — acceptable (Pages build is cheap; `concurrency: pages` already serializes).
- **API volume** — discovery is one batched GraphQL query per org; enrichment adds ~5–6 calls per repo
  (GraphQL + REST). For ~30 repos that's ~200 calls/day — well within the 5,000/hr REST and GraphQL budgets.
  Locally the enrichment loop takes a minute or two.
- **Coupling** — generating inside `update-doc.yml` couples content deploys to the API; the non-fatal step
  (§4.1) removes that risk.
- **Permission isolation** — collection is a distinct job, so any elevated read scope (e.g. the optional vuln
  count) is confined to it; `build-doc`/`deploy-doc` keep their existing minimal permissions.
- **Dynamic discovery** may surface unwanted repos → filter + manual exclude list.
- **External badge services** (shields, goreportcard, codecov) — cosmetic if temporarily down.
- **Workflow-filename overrides** — CI/CodeQL/release badge filenames are a hand-maintained table in the
  generator (§4.2). A repo that diverges without a table entry renders a broken badge until the table is
  updated; cosmetic. Auto-detection is a deferred nicety (§4.5).
- **`lint` metric depends on a convention** — lint status assumes a `lint`-named job in the CI ("Test")
  workflow; it degrades to `unknown` when that doesn't hold. Contributor counts cover real GitHub users only
  (anonymous/email-only authors are excluded), so the distinct totals are logins, not people-by-email.
- **Data is ephemeral** (not versioned) — by design; the generator script is the source of truth.

## 8. Out of scope (for now)

- **Historical trends / time-series** — would need a persistent store (a gist or a `data` branch); revisit if
  we want sparklines or week-over-week deltas.
- Per-repo drill-down pages.
- **Cross-repo event-driven triggers** — explicitly rejected as too complex; the daily cron is enough.

## 9. Rendering POC (2026-05-31) — results

Goal: de-risk **D-g** before building the generator — prove the Relearn theme renders data-driven tables
inside `tabs`. Approach: hand-crafted `dashboard.json` (3 repos: go-openapi/analysis, go-openapi/doc-site,
go-swagger/go-swagger) → custom shortcode + partials → local Hugo build.

**Key finding:** the theme's `tabs`/`tab` shortcodes are thin wrappers over `partial "shortcodes/tabs.html"`,
which accepts a **slice of tab dicts** (`title` + `content` HTML, emitted via `safeHTML`). A custom shortcode
builds that slice from data and calls the partial directly — no markup/JS replication. This is the mechanism.

**Verified in a real `hugo --minify` build** (`/dashboard/index.html`, clean build, no new warnings):
- 3 tabs (Status / Activity / Quality) with the theme's real nav buttons + `switchTab` JS + `tab-panel` markup.
- Per-org grouped rows (correct colspans), 15 badges rendering, `❗` threshold (>20 commits), `N/A` for
  unreleased repos, and the "Last updated" footer from `generatedAt`.

**Wrinkles surfaced** (both tracked: §4.2 / §7): assumed CI/CodeQL/release workflow filenames; CodeQL
default-setup has no workflow file.

**Local testing prerequisites:** Relearn theme fetched into `themes/hugo-relearn` (CI does this); `hugo`
present; the generator (next) needs `GH_TOKEN` / `gh auth`.
