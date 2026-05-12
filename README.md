# moorbrook/ci-workflows

Reusable GitHub Actions workflows for moorbrook Rust projects.

## Usage

In a caller repo, create `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref_name }}-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: true

jobs:
  fmt:
    uses: moorbrook/ci-workflows/.github/workflows/rust-fmt.yml@v1
  lint:
    uses: moorbrook/ci-workflows/.github/workflows/rust-lint.yml@v1
  test:
    uses: moorbrook/ci-workflows/.github/workflows/rust-test.yml@v1

  # Optional: gate cross-compile on main-push only (private repos: 2x quota multiplier).
  windows-cross:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: moorbrook/ci-workflows/.github/workflows/rust-windows-cross.yml@v1
```

## Available workflows

| File | Purpose | Inputs |
|---|---|---|
| `rust-fmt.yml` | `cargo fmt --check` | `package-selection` (default `--all`) |
| `rust-lint.yml` | `cargo clippy --all-targets -- -D warnings` | `package-selection`, `extra-args`, `clippy-allows` |
| `rust-test.yml` | `cargo nextest run` + doc tests | `package-selection`, `os-matrix`, `use-nextest`, `extra-args` |
| `rust-windows-cross.yml` | `cargo xwin build --target x86_64-pc-windows-msvc` | `package-selection`, `extra-args` |

### Vendored `[patch.crates-io]` path deps

If your workspace patches a crate with a vendored path dep (e.g. `lopdf =
{ path = "vendor/lopdf-patch" }`), Cargo treats that path the same as a
first-party crate: `cargo fmt --all` sweeps it, `cargo clippy --workspace`
lints it with no `--cap-lints=warn` shield, and `cargo test --workspace`
compiles its test target. Upstream-style drift, lints, or test fixtures
that aren't tracked in your repo will fail CI.

Workaround: pass an explicit package list via `package-selection` so the
patched dep is skipped:

```yaml
jobs:
  fmt:
    uses: moorbrook/ci-workflows/.github/workflows/rust-fmt.yml@v1
    with:
      package-selection: "-p my-crate -p other-crate"
  lint:
    uses: moorbrook/ci-workflows/.github/workflows/rust-lint.yml@v1
    with:
      package-selection: "-p my-crate -p other-crate"
  test:
    uses: moorbrook/ci-workflows/.github/workflows/rust-test.yml@v1
    with:
      package-selection: "-p my-crate -p other-crate"
```

## Cost discipline

- **All workflows default to `ubuntu-latest`** (1× quota multiplier).
- **Windows runners** are 2× quota — `rust-windows-cross.yml` uses ubuntu + cargo-xwin instead.
- **macOS runners are 10× quota.** Don't add macOS to `os-matrix` on private repos unless you have budget headroom.
- Callers should gate slow jobs on main-push only via `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`.

## Versioning

Pin to `@v1` for stability; the tag moves only on backward-compatible upgrades. Breaking changes ship as `@v2` so callers migrate on their own schedule.

For trunk testing: `@main` is the latest (may break).
