# actions

Org-wide reusable GitHub Actions workflows for the **external-systems** org:
Nix/NixOS CI on the org's self-hosted NixOS runners.

This repo is a **library, not a consumer**. Every workflow is triggered only by
`on: workflow_call` — there are no `push`, `pull_request`, or
`workflow_dispatch` triggers here, and this repo never runs itself. It contains
no secrets and no code that executes on the org's hardware.

## Security contract

The org's repos are public, and these workflows are the root of trust for CI:

- **Fork PRs never run on self-hosted runners.** Every job that executes repo
  code carries a baked-in guard
  (`github.event.pull_request.head.repo.full_name == github.repository`).
  **Do not remove it.**
- **Consumers must reference `@v1` (or a full commit SHA), never `@main`.**
  `@main` is mutable; the tag is the reviewed artifact.
- The org's Actions policy must allow actions from `external-systems/*` for
  these `uses:` references to resolve.
- Callers run with the **caller's** `GITHUB_TOKEN` and secrets; each reusable
  workflow declares the minimum `permissions:` it needs (`contents: read`, or
  `contents: write` for the tagging job).
- To use these workflows on GitHub-hosted runners, pass
  `runs-on: '["ubuntu-latest"]'` — but nix is **not** preinstalled there and
  the org standard is the self-hosted runner. Don't do this without a reason.

## Runners

Jobs target the org's self-hosted NixOS runners via the label set
`["self-hosted", "external-systems"]` (default for every workflow's `runs-on`
input). The runners are ephemeral (one registration per job, workdir wiped),
run nix builds through the system daemon with `accept-flake-config = true`,
expose `/dev/kvm` to the nix build sandbox, and have `nix`, `git`, `gh`, `jq`,
`yq`, `curl`, `wget`, `openssh`, `git-lfs` preinstalled on PATH.

**No setup actions are needed or wanted** — no Determinate installer, no
cachix action, nothing third-party except `actions/checkout` (pinned to
`@v6.0.2`). Workflows are plain `run:` steps over the baked-in toolset.

## Workflows

### `nix-flake-check.yml` — flake check

Runs `nix flake check` (default flags `--show-trace -L`). Gates everything.

```yaml
jobs:
  flake-check:
    uses: external-systems/actions/.github/workflows/nix-flake-check.yml@v1
```

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `runs-on` | string | no | `'["self-hosted", "external-systems"]'` | JSON array of runner labels |
| `flake-check-flags` | string | no | `"--show-trace -L"` | Flags passed verbatim to `nix flake check` |

Caller permissions needed: none beyond defaults (job uses `contents: read`).

### `nix-flake-check-and-tag.yml` — check gates rolling-semver tag

The flagship pipeline. Job 1 runs `nix flake check`; job 2 (gated on job 1)
tags the commit `0.<rolling-minor>.<commit-count-since-init>` and pushes the
tag. Idempotent: an existing tag is a no-op. Simultaneous pushes serialize via
a `concurrency` group; tag pushes do not retrigger anything (consumers filter
on branches, and `GITHUB_TOKEN` pushes emit no events).

```yaml
jobs:
  check-and-tag:
    uses: external-systems/actions/.github/workflows/nix-flake-check-and-tag.yml@v1
    with:
      rolling-minor: "3"
```

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `runs-on` | string | no | `'["self-hosted", "external-systems"]'` | JSON array of runner labels |
| `rolling-minor` | string | **yes** | — | Rolling semver minor; each repo picks its own |

Caller permissions needed: the workflow requests `contents: write` for the tag
job; callers on the default `GITHUB_TOKEN` permission model need no extra
config.

### `nix-build.yml` — build an arbitrary installable

```yaml
jobs:
  build:
    uses: external-systems/actions/.github/workflows/nix-build.yml@v1
    with:
      installable: .#packages.x86_64-linux.myPackage
```

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `installable` | string | **yes** | — | Flake installable (e.g. `.#packages.x86_64-linux.foo`) |
| `runs-on` | string | no | `'["self-hosted", "external-systems"]'` | JSON array of runner labels |
| `build-flags` | string | no | `"-L"` | Flags passed verbatim to `nix build` |

Caller permissions needed: none beyond defaults (`contents: read`).

### `nixos-machine-build.yml` — build a NixOS system toplevel

Thin wrapper that nests a call to `nix-build.yml` with
`.#nixosConfigurations.<host>.config.system.build.toplevel` (nesting depth
stays at 1; the internal `./` reference always matches the caller's ref of
this repo).

```yaml
jobs:
  build-kamrui-h1:
    uses: external-systems/actions/.github/workflows/nixos-machine-build.yml@v1
    with:
      host: kamrui-h1
```

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `host` | string | **yes** | — | NixOS configuration name |
| `runs-on` | string | no | `'["self-hosted", "external-systems"]'` | JSON array of runner labels |

Caller permissions needed: none beyond defaults (`contents: read`).

### `nixos-vm-test.yml` — run a NixOS VM test

Runs a VM test through the consumer flake's `vm-test` app:
`nix run .#vm-test -- <test>-<size>`. The step's exit code **is** the test
result; the job log is the complete driver log. With
`extra-args: "--driver"` the app writes artifacts (screenshots, junit.xml,
resumable VM state) to `/tmp/home-vm-tests/<name>` on the runner. KVM is
available, so tests run at full speed. `size` is validated in a step and fails
fast with a clear message — md/lg are for idle hardware; the runner is a
30 GiB / 8c16t desktop that is in use.

**VM tests must stay manual-dispatch-only in consumers** — never wire this
workflow into `push`/`pull_request` triggers.

```yaml
jobs:
  vm-test:
    uses: external-systems/actions/.github/workflows/nixos-vm-test.yml@v1
    with:
      test: my-service
      size: sm
```

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `test` | string | **yes** | — | Test base name (appended with `-<size>`) |
| `size` | string | no | `"sm"` | One of `sm`, `md`, `lg` (validated) |
| `extra-args` | string | no | `""` | Passed verbatim to the app (e.g. `--driver`) |
| `runs-on` | string | no | `'["self-hosted", "external-systems"]'` | JSON array of runner labels |

Caller permissions needed: none beyond defaults (`contents: read`).

## Consumer call-sites (`external-systems/home`)

```yaml
# flake check + rolling-semver tag (home is on rolling minor 3)
check-and-tag:
  uses: external-systems/actions/.github/workflows/nix-flake-check-and-tag.yml@v1
  with:
    rolling-minor: "3"

# NixOS machine toplevel build
build-machine:
  uses: external-systems/actions/.github/workflows/nixos-machine-build.yml@v1
  with:
    host: kamrui-h1

# arbitrary installable build
build-package:
  uses: external-systems/actions/.github/workflows/nix-build.yml@v1
  with:
    installable: .#packages.x86_64-linux.myPackage

# VM test (workflow_dispatch only, never automatic)
vm-test:
  uses: external-systems/actions/.github/workflows/nixos-vm-test.yml@v1
  with:
    test: my-service
    size: sm
```

## Versioning

This repo is tagged `v1` (a plain annotated git tag, moved only by a human
after review). Consumers pin `@v1` or a full SHA. Breaking changes get a new
major tag.
