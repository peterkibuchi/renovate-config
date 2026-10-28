# renovate-config

A shareable [Renovate](https://docs.renovatebot.com/) preset that keeps dependency
update PRs cheap on CI and cautious about fresh releases.

## Use

Point your repo's `.github/renovate.json` at it:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>peterkibuchi/renovate-config"]
}
```

Add repo-specific `packageRules` after the extends line as needed — or fork this repo
and tune your own copy. Validate any change with
`npx --package renovate -- renovate-config-validator --strict`.

## What it does, and why

- **`minimumReleaseAge: 1 day` + `internalChecksFilter: strict`** — PRs are only
  created once a release is a day old. This matches pnpm 11's default release-age
  gate (so PRs aren't born failing `pnpm install`) and gives the ecosystem time to
  catch yanked or malicious releases before they reach you.
- **Daily grouped non-major updates, automerged on green CI** — two groups (prod, and
  dev/tooling — CI managers like GitHub Actions and Dockerfiles, custom managers (such as
  the biome schema tracker), the `packageManager` pin, `engines` ranges and pnpm catalog
  entries ride with dev, since lone PRs of those kinds go stale behind their siblings.
  `engines` uses `rangeStrategy: replace`, so a range only changes when a release falls
  outside it, and Node floors are never raised just because a new Node shipped) on a before-6am
  schedule. Few PRs, few CI runs, nothing left "behind base" to churn.
- **Majors automerge on green too — but stay individual (never grouped) and soak 5
  days**: one breaking bump can't block unrelated updates, and a major's early patch
  releases are where regressions surface. CI is the sole gate here; a repo with a thin
  test suite should override with `"automerge": false` for majors.
- **`prConcurrentLimit: 3`** — at most three Renovate PRs are open at once, so a backlog
  arrives a few PRs at a time.
- **Monthly `lockFileMaintenance`, automerged on green** — before 6am on the first of the
  month, Renovate regenerates the lockfile so transitive dependencies move even when no
  direct dependency was bumped.
- **`rebaseWhen: auto`** — Renovate rebases behind-base branches when automerge is on
  or the repo requires up-to-date branches; otherwise only on conflict. Grouping is
  what keeps the rebase count low.
- **`osvVulnerabilityAlerts: true`** — adds the [OSV database](https://osv.dev) alongside GitHub's
  advisories, so a vulnerability reaches you even when GitHub hasn't ingested it yet. Requires the
  repo to have Dependabot *alerts* enabled (alerts only — automated fixes stay off; Renovate owns
  the PRs).
- **`vulnerabilityAlerts.minimumReleaseAge: 1 day`** — deliberately mirrors pnpm 11's own
  `minimumReleaseAge`. Renovate defaults this to `null` (no soak), which opens a CVE PR that pnpm
  then refuses to install: red for a day, same landing time, one wasted CI run, and a red PR that
  teaches you to ignore red. **If pnpm's default changes, change this with it.**
- **`vulnerabilityAlerts` automerge** — security fixes skip the schedule and merge on
  green.
