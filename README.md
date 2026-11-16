# renovate-config

The shared config that the owner's repos point at instead of copying: a
[Renovate](https://docs.renovatebot.com/) preset that keeps dependency update PRs cheap on CI
and cautious about fresh releases, a [Biome](https://biomejs.dev/) config, and one reusable CI
workflow. Each repo extends them rather than keeping its own copy, so a change here reaches
all of them.

## The Biome config

Install it from this repo (it isn't published to npm), then extend it:

```sh
pnpm add -D "@peterkibuchi/config@github:peterkibuchi/renovate-config"
```

```json
{
  "$schema": "./node_modules/@biomejs/biome/configuration_schema.json",
  "extends": ["@peterkibuchi/config/biome"]
}
```

Add only what is true of that repo after the `extends` line: its generated files under
`files.includes` (entries add to the shared list), `tailwind.stylesheet` in a monorepo, and
`overrides`. It needs Biome 2.6.7 or later. The lockfile pins the commit you installed;
`pnpm update @peterkibuchi/config` moves it, as does the monthly lockfile refresh below.

## The CI workflow

`.github/workflows/ci.yml` here runs a repo's gate: it installs with the repo's pnpm and
`.node-version`, then runs whichever of `check`, `lint:ws`, `knip`, `typecheck` and `test`
the repo's `package.json` defines, and `build` only when asked. A repo's own workflow is
then:

```yaml
name: CI

on:
  pull_request:
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  ci:
    uses: peterkibuchi/renovate-config/.github/workflows/ci.yml@main
    # with:
    #   build: true   # where CI is the build gate: a template, or a public product repo
```

Its check reports as `ci / ci`; require that name on `main`. A job only one repo needs (an
end-to-end run, a bundle check) goes beside this one in that repo's workflow. Callers follow
`@main`, as they follow this preset, and the preset leaves that reference unpinned.

## The Renovate preset

### Use

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

### What it does, and why

- **`minimumReleaseAge: 1 day` + `internalChecksFilter: strict`** — PRs are only
  created once a release is a day old. This matches pnpm 11's default release-age
  gate (so PRs aren't born failing `pnpm install`) and gives the ecosystem time to
  catch yanked or malicious releases before they reach you.
- **Daily grouped non-major updates, automerged on green CI** — two groups (prod, and
  dev/tooling — CI managers like GitHub Actions and Dockerfiles, `.nvmrc` and
  `.node-version`, custom managers (such as the biome schema tracker), the
  `packageManager` pin, `engines` ranges, pnpm catalog entries and overrides ride with
  dev, since lone PRs of those kinds go stale behind their siblings.
  `engines` uses `rangeStrategy: replace`, so a range only changes when a release falls
  outside it, and Node floors are never raised just because a new Node shipped) on a before-6am
  schedule. Few PRs, few CI runs, nothing left "behind base" to churn.
- **Majors automerge on green too — but stay individual (never grouped) and soak 5
  days**: one breaking bump can't block unrelated updates, and a major's early patch
  releases are where regressions surface. CI is the sole gate here; a repo with a thin
  test suite should override with `"automerge": false` for majors.
- **`helpers:pinGitHubActionDigests`**: every GitHub Action is pinned to a commit SHA, with
  its version kept as a comment that Renovate bumps. A tag can be moved to new code after you
  last read it; a SHA can't, so a moved tag can't slip new code into a workflow that holds secrets.
  The first run opens one pinning PR per repo, grouped with the dev updates.
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
- **nitro follows its prereleases** (`ignoreUnstable: false`): nitro 3 has shipped only betas and
  release candidates so far, and Renovate skips prereleases by default, so sites built on a beta
  stayed on it. Drop the rule once a stable 3.x ships.
- **The shared CI workflow stays on `@main`** (`pinDigests: false` for this repo's own
  workflow): a change to it reaches every repo's next run, as a change to this preset does,
  instead of arriving as a pinning PR in each repo.
- **`vulnerabilityAlerts` automerge** — security fixes skip the schedule and merge on
  green.
