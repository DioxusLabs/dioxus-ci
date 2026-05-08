# dioxus-ci

Reusable GitHub Actions and workflows for [Dioxus](https://dioxuslabs.com) projects.

Two layers:

- **Composite actions**: fine-grained building blocks you drop into your own workflow.
- **Reusable workflows**: opinionated, batteries-included flows you call with one `uses:` line.

Pin this repository by tag or commit SHA in production. The internal third-party actions used by these workflows are pinned to exact commit SHAs.

---

## Quick start

CI checks for a typical Dioxus app:

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request: { types: [opened, synchronize, reopened, ready_for_review], branches: [main] }
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
jobs:
  check:  { uses: ealmloff/dioxus-ci/.github/workflows/check.yml@main }
  test:   { uses: ealmloff/dioxus-ci/.github/workflows/test.yml@main }
  clippy: { uses: ealmloff/dioxus-ci/.github/workflows/clippy.yml@main }
  fmt:    { uses: ealmloff/dioxus-ci/.github/workflows/fmt.yml@main }
```

GitHub Pages deploys use one unprivileged web build and one privileged publisher. The build workflow always uploads a static artifact; the publisher deploys it as the main site for default-branch pushes, or under `pr-preview/pr-<N>/` for pull requests.

```yaml
# .github/workflows/web.yml
name: Web
on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main]
permissions:
  contents: read
concurrency:
  group: web-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
jobs:
  build:
    uses: ealmloff/dioxus-ci/.github/workflows/web-build.yml@main
    with:
      working-directory: app
      ssg: true
      features: fullstack
      base-path: ${{ github.event.repository.name }}
      dx-cli-version: 0.7.7
```

```yaml
# .github/workflows/pages.yml
name: Pages
on:
  workflow_run:
    workflows: ["Web"]
    types: [completed]
  pull_request:
    types: [closed]
permissions:
  actions: read
  contents: write
  pull-requests: write
concurrency:
  group: pages-${{ github.event.workflow_run.id || github.event.pull_request.number }}
  cancel-in-progress: false
jobs:
  publish:
    uses: ealmloff/dioxus-ci/.github/workflows/pages-publish.yml@main
```

---

## Composite actions

### `setup-dioxus`

Install the Rust toolchain and a Swatinem cache.

```yaml
- uses: ealmloff/dioxus-ci/actions/setup-dioxus@main
  with:
    toolchain: stable
    targets: wasm32-unknown-unknown
```

| input | default | description |
|---|---|---|
| `toolchain` | `stable` | Rust toolchain string passed to the pinned `dtolnay/rust-toolchain` action. |
| `components` | `""` | Comma-separated toolchain components. |
| `targets` | `""` | Comma-separated toolchain targets. |
| `cache` | `"true"` | Set to `"false"` to manage caching yourself. |
| `cache-all-crates` | `"true"` | Passed to `Swatinem/rust-cache`. |
| `cache-on-failure` | `"false"` | Passed to `Swatinem/rust-cache`. |
| `cache-directories` | `""` | Extra dirs to cache. |
| `cache-shared-key` | `""` | Passed to `Swatinem/rust-cache` `shared-key`. |
| `cache-provider` | `""` | Passed to `Swatinem/rust-cache` `cache-provider`. |

### `install-dioxus-cli`

Install the `dx` CLI via [`cargo-binstall`](https://github.com/cargo-bins/cargo-binstall).

```yaml
- uses: ealmloff/dioxus-ci/actions/install-dioxus-cli@main
  with:
    version: 0.7.7
```

| input | default | description |
|---|---|---|
| `version` | `latest` | Version of `dioxus-cli` to install, or `latest`. |
| `force` | `"true"` | Pass `--force` to `cargo binstall`. |

### `free-disk-space`

Reclaim roughly 10-20 GB on Ubuntu runners by removing pre-installed toolchains. Runs only on Linux runners.

```yaml
- uses: ealmloff/dioxus-ci/actions/free-disk-space@main
```

### `build-web`

Run `dx build --web` with typed inputs for the common flags. Returns the path to the built assets.

```yaml
- id: build
  uses: ealmloff/dioxus-ci/actions/build-web@main
  with:
    working-directory: app
    ssg: true
    features: fullstack
    base-path: my-app
- run: ls "${{ steps.build.outputs.output-path }}"
```

| input | default | description |
|---|---|---|
| `working-directory` | `.` | Where `dx build` runs. |
| `ssg` | `"false"` | When `"true"`, adds `--fullstack --ssg`. |
| `features` | `""` | Cargo features passed as one `--features` value. |
| `no-default-features` | `"false"` | When `"true"`, adds `--no-default-features`. |
| `release` | `"true"` | When `"true"`, adds `--release`. |
| `debug-symbols` | `"true"` | When `"false"`, adds `--debug-symbols false`. |
| `base-path` | `""` | Adds `--base-path <value>` when set. |
| `output-dir` | `""` | Explicit built asset directory to return instead of resolving the standard Dioxus output path. |

| output | description |
|---|---|
| `output-path` | Absolute path to the built web assets. |
| `crate-name` | Name of the built crate when auto-resolved. Empty when `output-dir` is used. |

### `spa-404-fallback`

Write a `404.html` next to your built site so any URL deep-links to the SPA. Optionally recognizes `<base>/<prefix>/pr-<N>/` and root-level `<prefix>/pr-<N>/` PR-preview subroutes.

```yaml
- uses: ealmloff/dioxus-ci/actions/spa-404-fallback@main
  with:
    output-dir: docs
    base-path: my-app
    pr-preview-prefix: pr-preview
```

| input | default | description |
|---|---|---|
| `output-dir` | required | Directory to write `404.html` into. |
| `base-path` | `""` | Base path the app is deployed at. |
| `pr-preview-prefix` | `""` | Recognize `<base>/<prefix>/pr-<N>/` and root-level `<prefix>/pr-<N>/` subroutes when set. |

---

## Reusable workflows

| workflow | purpose |
|---|---|
| `check.yml` | `cargo check` with typed package, feature, target, and resolver inputs. |
| `test.yml` | `cargo test` with typed package, feature, target, and test-selection inputs. |
| `clippy.yml` | `cargo clippy` with typed target/lint inputs and `-D warnings` by default. |
| `fmt.yml` | `cargo fmt --all -- --check` by default. |
| `docs.yml` | `cargo doc --workspace --no-deps --all-features --document-private-items` with denied warnings by default. |
| `web-build.yml` | Build a Dioxus web app and upload a deployable Pages artifact. |
| `pages-publish.yml` | Publish validated Pages artifacts to `gh-pages`, comment on PR previews, and remove previews when PRs close. |

Pages web builds time out after 45 minutes. Pages publish jobs time out after 15 minutes, and PR-preview cleanup after 10 minutes.

The cargo workflows expose typed inputs for every supported flag — use the per-workflow tables below. For desktop/webview apt packages, Firefox, or disk cleanup, compose a custom job and install those dependencies explicitly.

Note the `all-features` defaults: `check.yml`, `clippy.yml`, and `docs.yml` default to `true`; `test.yml` defaults to `false`. Pass `all-features: false` explicitly when feature combinations matter.

### `check.yml`

| input | default | description |
|---|---|---|
| `toolchain` | `stable` | Rust toolchain string. |
| `workspace` | `true` | Pass `--workspace`. |
| `package` | `""` | Newline-separated package names passed with `--package`. |
| `exclude` | `""` | Newline-separated package names passed with `--exclude`. |
| `features` | `""` | Space-separated cargo features. |
| `all-features` | `true` | Pass `--all-features`. |
| `no-default-features` | `false` | Pass `--no-default-features`. |
| `target` | `""` | Adds `--target <value>`. |
| `all-targets` | `false` | Pass `--all-targets`. |
| `bins` | `false` | Pass `--bins`. |
| `examples` | `false` | Pass `--examples`. |
| `tests` | `false` | Pass `--tests`. |
| `benches` | `false` | Pass `--benches`. |
| `locked` | `false` | Pass `--locked`. |
| `frozen` | `false` | Pass `--frozen`. |
| `offline` | `false` | Pass `--offline`. |
| `jobs` | `""` | Adds `--jobs <value>`. |
| `cache-directories` | `""` | Extra dirs passed to `Swatinem/rust-cache`. |
| `runs-on` | `ubuntu-latest` | Runner label. |

### `test.yml`

| input | default | description |
|---|---|---|
| `toolchain` | `stable` | Rust toolchain string. |
| `workspace` | `true` | Pass `--workspace`. |
| `package` | `""` | Newline-separated `--package` names. |
| `exclude` | `""` | Newline-separated `--exclude` names. |
| `features` | `""` | Space-separated cargo features. |
| `all-features` | `false` | Pass `--all-features`. |
| `no-default-features` | `false` | Pass `--no-default-features`. |
| `target` | `""` | Adds `--target <value>`. |
| `test-name` | `""` | Optional cargo test filter. |
| `lib` | `false` | Pass `--lib`. |
| `bins` | `false` | Pass `--bins`. |
| `examples` | `false` | Pass `--examples`. |
| `tests` | `false` | Pass `--tests`. |
| `benches` | `false` | Pass `--benches`. |
| `all-targets` | `false` | Pass `--all-targets`. |
| `doc` | `false` | Pass `--doc`. |
| `no-run` | `false` | Pass `--no-run`. |
| `no-fail-fast` | `false` | Pass `--no-fail-fast`. |
| `ignored` | `false` | Pass `--ignored` after `--`. |
| `include-ignored` | `false` | Pass `--include-ignored` after `--`. |
| `locked` | `false` | Pass `--locked`. |
| `frozen` | `false` | Pass `--frozen`. |
| `offline` | `false` | Pass `--offline`. |
| `jobs` | `""` | Adds `--jobs <value>`. |
| `cache-directories` | `""` | Extra dirs passed to `Swatinem/rust-cache`. |
| `runs-on` | `ubuntu-24.04` | Runner label. |

### `clippy.yml`

| input | default | description |
|---|---|---|
| `toolchain` | `stable` | Rust toolchain string. |
| `workspace` | `true` | Pass `--workspace`. |
| `package` | `""` | Newline-separated `--package` names. |
| `exclude` | `""` | Newline-separated `--exclude` names. |
| `features` | `""` | Space-separated cargo features. |
| `all-features` | `true` | Pass `--all-features`. |
| `no-default-features` | `false` | Pass `--no-default-features`. |
| `target` | `""` | Adds `--target <value>`. |
| `all-targets` | `true` | Pass `--all-targets`. |
| `bins` | `false` | Pass `--bins`. |
| `examples` | `true` | Pass `--examples`. |
| `tests` | `true` | Pass `--tests`. |
| `benches` | `false` | Pass `--benches`. |
| `deny-warnings` | `true` | Append `-D warnings` after `--`. |
| `deny` | `""` | Newline-separated lints passed after `--` as `-D <lint>`. |
| `warn` | `""` | Newline-separated lints passed after `--` as `-W <lint>`. |
| `allow` | `""` | Newline-separated lints passed after `--` as `-A <lint>`. |
| `locked` | `false` | Pass `--locked`. |
| `frozen` | `false` | Pass `--frozen`. |
| `offline` | `false` | Pass `--offline`. |
| `jobs` | `""` | Adds `--jobs <value>`. |
| `cache-directories` | `""` | Extra dirs passed to `Swatinem/rust-cache`. |
| `runs-on` | `ubuntu-24.04` | Runner label. |

### `fmt.yml`

| input | default | description |
|---|---|---|
| `toolchain` | `stable` | Rust toolchain string. |
| `all` | `true` | Pass `--all`. |
| `package` | `""` | Newline-separated `--package` names. |
| `check` | `true` | Pass `-- --check`. |
| `runs-on` | `ubuntu-24.04` | Runner label. |

### `docs.yml`

Defaults to `nightly` so `--document-private-items` works on the standard library docs.

| input | default | description |
|---|---|---|
| `toolchain` | `nightly` | Rust toolchain string. |
| `workspace` | `true` | Pass `--workspace`. |
| `package` | `""` | Newline-separated `--package` names. |
| `exclude` | `""` | Newline-separated `--exclude` names. |
| `features` | `""` | Space-separated cargo features. |
| `all-features` | `true` | Pass `--all-features`. |
| `no-default-features` | `false` | Pass `--no-default-features`. |
| `target` | `""` | Adds `--target <value>`. |
| `no-deps` | `true` | Pass `--no-deps`. |
| `document-private-items` | `true` | Pass `--document-private-items`. |
| `deny-warnings` | `true` | Set `RUSTDOCFLAGS=-D warnings`. |
| `open` | `false` | Pass `--open`. |
| `locked` | `false` | Pass `--locked`. |
| `frozen` | `false` | Pass `--frozen`. |
| `offline` | `false` | Pass `--offline`. |
| `jobs` | `""` | Adds `--jobs <value>`. |
| `cache-directories` | `""` | Extra dirs passed to `Swatinem/rust-cache`. |
| `runs-on` | `ubuntu-24.04` | Runner label. |

### `web-build.yml`

Builds a Dioxus web app and uploads a deployable Pages artifact (with metadata that `pages-publish.yml` validates).

| input | default | description |
|---|---|---|
| `working-directory` | `.` | Where `dx build` runs. |
| `ssg` | `false` | When `true`, adds `--fullstack --ssg`. |
| `features` | `""` | Cargo features passed as one `--features` value. |
| `no-default-features` | `false` | When `true`, adds `--no-default-features`. |
| `debug-symbols` | `true` | When `false`, adds `--debug-symbols false`. |
| `base-path` | `""` | Adds `--base-path <value>`. PR previews are built under `<base-path>/pr-preview/pr-<N>`. |
| `artifact-retention-days` | `7` | Retention for the uploaded artifact. |
| `dx-cli-version` | `latest` | Version of `dioxus-cli` to install. |
| `toolchain` | `stable` | Rust toolchain string. |
| `runs-on` | `ubuntu-latest` | Runner label. |
| `submodules` | `""` | Forwarded to `actions/checkout` `submodules`. |
| `cache-provider` | `""` | Forwarded to `Swatinem/rust-cache` `cache-provider`. |

### `pages-publish.yml`

Wire it from a separate workflow file using `workflow_run` (for publish) and `pull_request: closed` (for cleanup) — see the [Pages quick-start](#quick-start) above.

| input | default | description |
|---|---|---|
| `pages-url-base` | `""` | Site root used for PR-preview comments. Defaults to `https://<owner>.github.io`. |
| `branch` | `gh-pages` | GitHub Pages branch to deploy to. |
| `runs-on` | `ubuntu-latest` | Runner label. |

---

## Recipes

### Use a custom toolchain

```yaml
jobs:
  check:
    uses: ealmloff/dioxus-ci/.github/workflows/check.yml@main
    with:
      toolchain: 1.88.0
```

### Compile Linux desktop dependencies

```yaml
jobs:
  desktop-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5
      - run: |
          sudo apt-get update
          sudo apt-get install -y libwebkit2gtk-4.1-dev libgtk-3-dev libayatana-appindicator3-dev libxdo-dev
      - uses: ealmloff/dioxus-ci/actions/setup-dioxus@main
      - run: cargo check --workspace
```

### Compose your own deploy job

```yaml
jobs:
  custom-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5
      - uses: ealmloff/dioxus-ci/actions/setup-dioxus@main
        with: { targets: wasm32-unknown-unknown, cache-directories: target/dx }
      - uses: ealmloff/dioxus-ci/actions/install-dioxus-cli@main
      - id: build
        uses: ealmloff/dioxus-ci/actions/build-web@main
        with: { working-directory: app, ssg: true, features: fullstack, base-path: my-app }
      - uses: ealmloff/dioxus-ci/actions/spa-404-fallback@main
        with: { output-dir: '${{ steps.build.outputs.output-path }}', base-path: my-app }
      # Now deploy `${{ steps.build.outputs.output-path }}` wherever you like.
```

---

## License

Dual-licensed under [MIT](LICENSE) at your option, matching the rest of the Dioxus ecosystem.
