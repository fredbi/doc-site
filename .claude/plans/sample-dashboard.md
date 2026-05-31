---
title: go-openapi dashboard
description: All our repos at a glance
---

## Dashboard

All our repos at a glance.


Platforms we rely upon:

| Service       | go-openapi | go-swagger |
|---------------|------------|------------|
| Community Chat|[![Discord Channel][discord-badge]][discord-url]
| SCM           |[Github][github-goopenapi]|[Github][github-goswagger]|
| Badges        |<td colspan=2>[Shields.io][shields.io]</td>|
| Documentation |<td colspan=2>[pkg.go.dev][pkggodev]</td>|
| Documentation |[github pages][https://go-openapi.github.io/doc-site][[goswagger.io][https://goswagger.io]|
|<td colspan=3>Quality</td>|
| Code Quality  |[CodeFactor][codefactor]
| Code Quality  |[GoReportCard][goreport-goopenapi]
| Test coverage |[CodeCov][codecov-goopenapi]
|<td colspan=3>Shipping</td>|
| Linux packages|N/A|[CloudSmith][]
| Docker images |N/A|[Quay.io][]|

{{< tabs >}}
<!-- exclude special repo .github from list -->
{{% tab title="Status" %}}
<!-- Synthetic dashboard to see all non-archived repos with badges and links -->

| Repo | Release | CI | Last updated | Commits since last release | Cut a release   |
|------|---------|----|--------------|----------------------------|-----------------|
|<td colspan=6>go-openapi</td>|
|[analysis][analysis]|v0.25.1|[![Tests][analysis-test-badge]][analysis-test-url]|May 31 2026|![Commits since latest release][analysis-commits-badge]|[🚀][analys-cut-release]|
|[doc-site][doc-site]|N/A|N/A|May 30 2026|N/A|N/A|
|<td colspan=6>go-swagger</td>|
[[go-swagger][go-swagger]|||||N/A|
[[examples][examples]||||||<!-- no releases -->

> Legend
>
> * Release: latest github release. Past releases may be only tag releases. Some repos (examples, doc-site) are not released.
> * CI: latest CI status on master branch
> * Commits: all commits since last release, including bots and merge commits
> * Cut a release: bump release job (some repos may not be equipped with it yet).
{{% /tab %}}

{{% tab title="Quality" %}}

| Repo | Test cov. | CodeFactor | GoReportCard | CodeQL | Linter config | Lint status |
|------|-----------|------------|--------------|--------|---------------|-------------|
|<td colspan=6>go-openapi</td>|
|[analysis][analysis][![Coverage][analysis-cov-badge]][analysis-cov-url]|[![CodeFactor Grade][analysis-codefactor-badge]][analysis-codefactor-url]|[![Go Report Card][analysis-gocard-badge]][analysis-gocard-url]|[][analysis-golangci.yml]|[![Lint status][analysis-lint-badge]][analysis-lint-url]|
|<td colspan=6>go-swagger</td>|

> Legend
>
> * Test cov.: test coverage measured by codecov.io. Maintains history and trends.
> * CodeFactor: independent code quality assessement. Usually simpler than our lint. Usually stricter on things like complex functions.
> * GoReportCard: independent code quality assessement. Lightweight scan with the few things that always matter. Should be almost always 100%.
> * CodeQL: independent code quality assessment by github CodeQL (latest run status on master).
> * Linter: link to current linter configuration. Some repos may have a slightly different config or thresholds.
> * Lint status: latest status of the go lint job on master
{{% /tab %}}

{{% tab title="Activity" %}}
| Repo | Pending PRs | Pending issues | Commits since last release | Commits MTD | Commits MTD excl. bots | Commits YTD | Commits YTD excl. bots | Releases YTD | Total commits | Total contributors | Cut a release |
|------|---------|------------|----|----------|
|<td colspan=6>go-openapi</td>|
|[analysis][analysis]|3|1|40❗|5|1|45|4|6|1206|[🚀][analys-cut-release]| <!-- 40 commits since last release is > 20: show with "!" -->
|<td colspan=6>go-swagger</td>|
[[go-swagger][go-swagger]|||||N/A|
{{% /tab %}}

{{% tab title="Go" %}}
<!-- Go-specific information -->
| Repo | Release | go version | go doc | Is mono-repo | Top languages |
|------|---------|------------|--------|--------------|--------------|
|<td colspan=6>go-openapi</td>|
|[analysis][analysis]|v0.25.1|[![go version][analysis-goversion-badge]][analysis-goversion-url][![GoDoc][analysis-godoc-badge]][analysis-godoc-url]|No|![Top language][analysis-top-badge]|
|<td colspan=6>go-swagger</td>|
[[go-swagger][go-swagger]|||||
[[examples][examples]|||||

> Legend
>
> * go version: the minimum go version requirement in the root go.mod
> * go doc: link to the public pkg.go.doc documentation site (latest release).
> * is mono-repo: some of our repos are organized as go mono-repo, with multiple go.mod files
> * top languages: reminds us when other languages coexist with go in a repo

{{% /tab %}}

{{% tab title="Github" %}}
| Repo | Tags | Desc. | License | Default Branch | Protected Master| Check status | Forks | Stars |
|------|------|-------|---------|-----------------|------------------|--------------|-------|-------|
|[analysis][analysis]|`openapi`,`swagger2`|openapi specification object model analyzer|[![License][analysis-license-badge]][analysis-license-url]|master|Yes|Yes|23|1025|

> Legend
>
> * Tags: github tags for repo discovery
> * Desc.: github repo description
> * License: all our repos are licensed under the Apache 2.0 terms
> * Default branch: all our repos are expected to use "master" as their default branch (not "main") 
> * Protected: all our repos are expected to have master protected
> * Check status: all our repos are expected to enforce CI status checks on Pull Requests

**Archived repos**

| Repo | Tags | Desc. | Archived on |
|------|------|-------|-------------|
|<td colspan=4>go-openapi</td>|
|[spec3][spec3]|`openapi`,`swagger2`|openapi specification object model analyzer|2025-11-09|
|<td colspan=4>go-swagger</td>|

{{% /tab %}}

<!-- Security to be defined
{{% tab title="Security" %}}
| Repo | Policy | Dependabot update | Open vulnerabilities | ... |
|------|------|-------|---------|------------|----|----------|
| [analysis][analysis] | [Policy][analysis-security-policy] | ...|

{{% /tab %}}
-->

<!-- go-swagger specific for now: docker and package manager repos checks
{{% tab title="Shipping" %}}
{{% /tab %}}
-->
{{% /tabs %}}


<!-- references -->
[discord-badge]: https://img.shields.io/discord/1446918742398341256?logo=discord&label=discord&color=blue
[discord-url]: https://discord.gg/FfnFYaC3k5
[github-goopenapi]: github.com/go-openapi/
[github-goswagger]: github.com/go-swagger/
[codefactor]: https://www.codefactor.io/dashboard
[codecov]: https://app.codecov.io/github/go-openapi
[codecov-goswagger]: https://app.codecov.io/github/go-swagger

<!-- go-openapi/analysis -->
[analysis]: https://github.com/go-openapi/analysis
[analysis-goversion-badge]: https://img.shields.io/github/go-mod/go-version/go-openapi/analysis
[analysis-goversion-url]: https://github.com/go-openapi/analysis/blob/master/go.mod
[analysis-test-badge]: https://github.com/go-openapi/analysis/actions/workflows/go-test.yml/badge.svg
[analysis-test-url]: https://github.com/go-openapi/analysis/actions/workflows/go-test.yml
[analysis-cov-badge]: https://codecov.io/gh/go-openapi/analysis/branch/master/graph/badge.svg
[analysis-cov-url]: https://codecov.io/gh/go-openapi/analysis
[analysis-security-policy]: https://github.com/go-openapi/analysis#security-ov-file
[analysis-license-badge]: http://img.shields.io/badge/license-Apache%20v2-orange.svg
[analysis-license-url]: https://github.com/go-openapi/analysis/?tab=Apache-2.0-1-ov-file#readme
[analysis-top-badge]: https://img.shields.io/github/languages/top/go-openapi/analysis
[analysis-commits-badge]: https://img.shields.io/github/commits-since/go-openapi/analysis/latest
[analysis-cut-release]: https://github.com/go-openapi/analysis/actions/workflows/bump-release.yml
[analysis-golangci.yml]: https://github.com/go-openapi/analysis/blob/master/.golangci.yml

<!-- go-openapi archived repos -->
[spec3]: https://codecov.io/gh/go-openapi/spec3

<!-- go-swagger/go-swagger -->
[go-swagger]: https://github.com/go-swagger/go-swagger

<!-- go-swagger/examples -->
[examples]: https://github.com/go-swagger/examples

