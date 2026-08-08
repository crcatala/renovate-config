# renovate-config

Shared Renovate policy for the fleet. One place to change dependency policy for
every repo instead of fifteen.

## Use

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>crcatala/renovate-config"]
}
```

That resolves to `default.json` on this repo's default branch, **at run time**.
There is no version to bump: edit this repo and every consumer picks the change
up on its next Renovate run.

To pin — or to canary a change against one repo before merging it here — add a
ref suffix:

```json
"extends": ["github>crcatala/renovate-config#my-branch"]
```

This repo is **public** deliberately. Preset resolution has to work from the
private repos in the fleet too, and a public preset avoids having to install the
Renovate app here just to read policy that contains no secrets.

## Why there is only one preset

There is no `js.json`, `python.json` or `cli.json`, and that is on purpose.

Renovate package rules are **inert when they do not match**. A rule scoped to
`matchManagers: ["pep621"]` does nothing in a repo with no Python, and costs
nothing to carry. So one preset can hold rules for npm, Bundler, Python, Go,
Rust, containers and GitHub Actions at once, and each repo silently gets only
the ones that apply to it.

Splitting by language would invert that: every new language would mean a new
preset file **and** a PR against every repo that should use it. Keeping one
preset means a new language is one edit here, and every repo that later grows
that language is already covered.

Rules already ship for managers no repo currently uses (Go, Rust, containers).
That is the point — the first Go repo needs no config change.

Split this only when **policy** genuinely differs, not when language does. So
far it does not.

## What is deliberately not here

Anything genuinely repo-specific stays in the repo's own `renovate.json`:

- `customManagers` for repo-specific version annotations, e.g. `momentium-app`
  parsing `ARG NODE_VERSION` out of its Dockerfile
- a shorter `minimumReleaseAge` where a repo wants faster updates —
  `momentium-app` and `shielded-sh` run 3 days rather than the fleet's 7
- `prConcurrentLimit` and similar per-repo throughput knobs

A repo overrides by setting the key after the `extends`; later config wins.

## Notes on the policy

**`minimumReleaseAgeBehaviour: "timestamp-optional"` is load-bearing.** Renovate
42 (2025-11-06) changed this default to `timestamp-required`, which holds an
update **indefinitely** when its datasource reports no release timestamp —
rather than failing it. Two common update types have no timestamp: GitHub
Actions pinned to commit SHAs (a SHA is not a published release) and lockfile
maintenance (which adopts no version at all).

The result was seven PRs stuck across six repos, the oldest since 2026-01-26,
all green and all unmergeable, waiting on a `renovate/stability-days` status
that could never resolve. `timestamp-optional` keeps the cooldown enforced
everywhere a timestamp exists — npm and RubyGems, where it was always doing the
real work — and lets undatable updates through instead of parking them.

**Lockfile maintenance is monthly, not weekly.** A refresh is a large diff
(~4,600 lines in one recent case) that is not reviewable line by line. One
batched PR a month earns real attention; a weekly one gets rubber-stamped.

**Major updates never automerge**, in any ecosystem. `0.x` minors do not either,
since a 0.x minor is a breaking change by semver convention.

**Lifecycle scripts are ignored** (`ignoreScripts`), because install and
postinstall hooks are the usual vector for npm supply-chain attacks.
