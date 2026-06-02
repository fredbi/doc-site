# Plan — Dashboard "Github" tab: security metrics

> **Status:** ✅ implemented & validated locally (2026-06-02), branch
> `feat/dashboard-sec` — committed, not yet pushed. Phase-2 extension of
> `repo-dashboard.md` (resolves **D-h**, the deferred open-vulnerability metric).
> Authored 2026-06-02 (with Claude).
>
> Live-data validation: alerts — analysis/loads/runtime 6, go-swagger 1; reports
> — swag 2, spec 3, go-swagger 4 (with a `repo`-scoped token; CI uses per-org
> go-openapi-bot App tokens). UX: a leading `⚠️` precedes the alert count when
> >0, and stands alone when a source is unreadable. **Remaining:** confirm the
> App grants on both org installations include *repository security advisories:
> read* (done by Fred 2026-06-02), then push and let CI run.

## Context

The repository status dashboard (Github tab) currently shows administrative
metadata only. Maintainers have no at-a-glance view of *security posture* across
the ~30 go-openapi + go-swagger repos. We add two columns to the **Github** tab:

1. **Security alerts** — a combined count of *open* security alerts across all
   flavors (code scanning + Dependabot + secret scanning). Non-zero → link to
   `https://github.com/{owner}/{repo}/security`.
2. **Security reports** — *existence only* (⚠️) of open repository security
   advisories (privately-reported / draft vulnerabilities). >0 → ⚠️ linking to
   `https://github.com/{owner}/{repo}/security/advisories`.

**Flag-anything intent:** a real alert *or* a read failure ("tool failure", 403,
5xx) must both surface, so a broken/forbidden scanner never looks like a clean
"0". A flavor we cannot read is rendered as ⚠️ **unknown**, distinct from a
genuine zero.

### Key constraint discovered during research

These security APIs are **not public**. The dashboard runs from `doc-site` but
reads *other* repos' data, so:

- The job-level `security-events: read` / `security_reports: read` added to
  `update-doc.yml` are **inert for this feature** — they only scope doc-site's
  own `GITHUB_TOKEN`, not cross-repo reads. `security_reports` is also **not a
  valid Actions permission key**. Both are removed (see §4).
- Cross-org security reads require a **go-openapi-bot GitHub App installation
  token** (token exchange via the shared gh-action), with the App **installed on
  both orgs** and granted read on: *Code scanning alerts, Dependabot alerts,
  Secret scanning alerts, Repository security advisories*. This token is used
  **only** for the four security calls; `github.token` still does all existing
  public-data reads.

## Decisions

- **Alert column:** one **combined count** (no per-flavor count breakdown). The
  ⚠️ unknown marker carries a `title` naming which flavor(s) failed.
- **Unknown vs zero:** flavors we can read are summed; any flavor that errors
  adds a ⚠️ marker. So `0` = all flavors read clean, none open; `⚠️` = at least
  one flavor unreadable; `N` / `N ⚠️` = N open (some readable, maybe one failed).
- **Token:** go-openapi-bot **App installation token** via the shared gh-action
  (no PAT). Passed to the generator as `SECURITY_TOKEN`.

## 1. Generator — `hack/doc-site/dashboard/collect-dashboard.sh`

Follows the existing **non-fatal helper** pattern (each helper prints a value on
stdout, never aborts). Two new helpers + four schema fields, wired into the
phase-2 enrichment loop (archived repos keep neutral defaults, as today).

- **1a. Token plumbing:** `SECURITY_TOKEN="${SECURITY_TOKEN:-${GH_TOKEN:-}}"`;
  security helpers run their `gh` calls under `GH_TOKEN="${SECURITY_TOKEN}"`.
  Unset/unprivileged → helpers return `unknown`, columns degrade to ⚠️ (non-fatal).
- **1b. `security_alerts()`** → `{count, unknown:[flavors]}`. Per flavor
  (`code-scanning` / `dependabot` / `secret-scanning`, `state=open&per_page=100`,
  paginated): 2xx → add length; *disabled* (error `.message` matches
  `disabled|not enabled|no analysis|advanced security|not available`) → 0; any
  other error → flavor marked unknown.
- **1c. `security_reports()`** → `{count, unknown:bool}`.
  `/security-advisories?per_page=100`, count `state` ∈ {`triage`,`draft`}; same
  disabled-vs-error treatment.
- **1d. Schema:** add to `NODE_TO_REPO` defaults and the enrichment merge:
  `securityAlerts:0, securityAlertsUnknown:[], securityReports:0, securityReportsUnknown:false`.

## 2. Template — `layouts/partials/dashboard/github.html`

Two `<th>` (Security alerts, Security reports) at the end of the header; group
`colspan` 7 → 9. **Alerts cell** (`/security`): `>0` → `N`(+`⚠️` if unknown);
`0`+unknown → `⚠️` (title names flavors); else `0`/dash. **Reports cell**
(`/security/advisories`): `>0` → `⚠️` (title "N open advisories"); unknown → `⚠️`;
else dash. Links keep `target="_blank" rel="noopener"`. Archived table untouched.

## 3. Docs

- `hack/doc-site/dashboard/README.md` — metrics + `SECURITY_TOKEN` + App grants.
- `docs/doc-site/dashboard/_index.md` — legend for the two columns.
- `repo-dashboard.md` — resolve **D-h**, tick the Security item.

## 4. CI — `.github/workflows/update-doc.yml` (`collect-dashboard`)

Remove `security-events: read` + invalid `security_reports: read` (keep
`contents: read`). Add a step minting the go-openapi-bot App installation token
(shared `go-openapi/gh-actions` action), pass it as `SECURITY_TOKEN`; keep
`GH_TOKEN` unchanged. Job stays `continue-on-error: true`.

## 5. Verification (local — the token pause)

Provide `SECURITY_TOKEN` (App installation token, or throwaway PAT `repo` +
`security_events`); confirm the App is on both orgs with the four read grants.
Then: probe raw APIs with `gh api -i` to finalize the disabled-vs-error message
strings + advisory `state` behavior; run the generator and `jq` the four new
fields across repos of each shape (clean-0 / has-alerts / has-advisory /
flavor-unknown); render via the Hugo dev server and check links, ⚠️, and that the
group-header colspans line up at 9.

## Out of scope
- Per-flavor count columns beyond the ⚠️ title.
- A standalone "Security" tab (deferred — `repo-dashboard.md` §4.5).
- Branch-protection / required-checks (separate Github-tab phase-2 item).
