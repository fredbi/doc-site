---
title: go-openapi dashboard
description: All our repos at a glance
---

## Dashboard

All our repos at a glance.

[![Discord Channel][discord-badge]][discord-url]

{{< tabs >}}
{{% tab title="Status" %}}
<!-- Synthetic dashboard to see all non-archived repos with badges and links -->

| Repo | Release | go version | CI | Last updated | Is mono-repo |
|------|---------|------------|----|--------------|--------------|
|<td colspan=6>go-openapi</td>|
|[analysis][analysis] | v0.25.1 | [![go version][analysis-goversion-badge]][analysis-goversion-url] |[![Tests][analysis-test-badge]][analysis-test-url]|[![Coverage][analysis-cov-badge]][analysis-cov-url]
|<td colspan=6>go-swagger</td>|
[[go-swagger][go-swagger]|||||
[[examples][examples]|||||

{{% /tab %}}

{{% tab title="Quality" %}}

| Repo | Test cov |CodeFactor|GoReportCard|
|------|---------|----------|----------|------------|
|<td colspan=6>go-openapi</td>|
| [analysis][analysis] | v0.25.1|[![Coverage][analysis-cov-badge]][analysis-cov-url]|||
|<td colspan=6>go-swagger</td>|

{{% /tab %}}

{{% tab title="Activity" %}}
| Repo | Pending PRs | Pending issues | Commits since last release | Commits MTD | Commits MTD excl. bots | Commits YTD | Commits YTD excl. bots | Releases YTD | Total commits | Total contributors |
|------|---------|------------|----|----------|
| [analysis][analysis] | | ||
{{% /tab %}}

{{% tab title="Github" %}}
| Repo | Tags | Desc. | License | Security | Protected Master | Check status | Forks | Stars |
|------|------|-------|---------|----------|------------------|--------------|-------|-------|
| [analysis][analysis] | `openapi`,`swagger2` |openapi specification object model analyzer| v0.25.1 | [![go version][analysis-goversion-badge]][analysis-goversion-url] |

**Archived repos**
Repo | Tags | Desc. | License | Security | Protected Master | Check status | ... |
|------|------|-------|---------|------------|----|----------|
| [spec3][spec3] | `openapi`,`swagger2` |openapi specification object model analyzer| v0.25.1 | [![go version][analysis-goversion-badge]][analysis-goversion-url] |

{{% /tab %}}

{{% tab title="Security" %}}
| Repo | Policy | Github checks | Dependabot | Protected Master | Check status | ... |
|------|------|-------|---------|------------|----|----------|
| [analysis][analysis] | `openapi`,`swagger2` |openapi specification object model analyzer| v0.25.1 | [![go version][analysis-goversion-badge]][analysis-goversion-url] |

{{% /tab %}}

{{% /tabs %}}


<!-- references -->
[discord-badge]: https://img.shields.io/discord/1446918742398341256?logo=discord&label=discord&color=blue
[discord-url]: https://discord.gg/FfnFYaC3k5

<!-- go-openapi/analysis -->
[analysis]: https://github.com/go-openapi/analysis
[analysis-goversion-badge]: https://img.shields.io/github/go-mod/go-version/go-openapi/analysis
[analysis-goversion-url]: https://github.com/go-openapi/analysis/blob/master/go.mod
[analysis-test-badge]: https://github.com/go-openapi/analysis/actions/workflows/go-test.yml/badge.svg
[analysis-test-url]: https://github.com/go-openapi/analysis/actions/workflows/go-test.yml
[analysis-cov-badge]: https://codecov.io/gh/go-openapi/analysis/branch/master/graph/badge.svg
[analysis-cov-url]: https://codecov.io/gh/go-openapi/analysis

<!-- go-openapi archived repos -->
[spec3]: https://codecov.io/gh/go-openapi/spec3

<!-- go-swagger/go-swagger -->
[go-swagger]: https://github.com/go-swagger/go-swagger

<!-- go-swagger/examples -->
[examples]: https://github.com/go-swagger/examples

