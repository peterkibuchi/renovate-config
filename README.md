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
- **`group:allNonMajor` + weekly schedule** — one grouped PR for patch/minor updates
  instead of dozens. Fewer PRs, fewer CI runs, nothing left "behind base" to churn.
- **`rebaseWhen: conflicted`** — merging one PR doesn't trigger CI reruns on every
  other open PR. Your CI on the default branch still catches bad combinations.
- **Non-major updates automerge on green CI; majors get individual PRs and a human.**
- **`vulnerabilityAlerts` automerge** — security fixes don't wait for the weekly
  window.

- **`rebaseWhen: auto`** — Renovate rebases behind-base branches when automerge is on or the
  repo requires up-to-date branches; otherwise only on conflict. Grouping keeps the count low.
