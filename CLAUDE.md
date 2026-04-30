# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Bazel-Gazelle: a build-file generator for Bazel. It creates and updates `BUILD.bazel` files. Native support for Go and Protocol Buffers; extensible via the `language.Language` interface for other languages. Gazelle is also invoked internally by the `go_repository` rule to generate build files for external repos.

This repo is the upstream `github.com/bazelbuild/bazel-gazelle`. It is **not** part of the surrounding `~/code/` EnthuZiastic monorepo — ignore the parent `CLAUDE.md` files about platform/admin/consumer services. This is a vendored upstream OSS dependency.

## Build, test, run

All workflows go through Bazel.

| Task | Command |
|---|---|
| Build everything | `bazel build //...` |
| Run all tests | `bazel test //...` |
| Run Gazelle on this repo | `bazel run //:gazelle` |
| CI parity check (BUILD files up-to-date) | `bazel run //:gazelle_ci` — runs `gazelle fix --mode=diff`; non-zero exit = drift |
| Run a single test | `bazel test //path/to/pkg:target_test` (e.g. `//cmd/gazelle:gazelle_test`, `//language/go:go_test`) |
| Race-detector tests | `bazel test --@io_bazel_rules_go//go/config:race //...` |
| Bzlmod mode (default Bazel 7+) | plain `bazel test //...` |
| Legacy WORKSPACE mode | `bazel test --noenable_bzlmod --enable_workspace //...` |

CI matrix lives in `.bazelci/presubmit.yml` (BuildKite) and runs Ubuntu/macOS/Windows × Bazel 7/8/9 × bzlmod and WORKSPACE. Two BCR test modules under `tests/bcr/go_work` and `tests/bcr/go_mod` exercise consumer-side behavior. Mirror its target lists when reproducing CI failures locally.

After editing Go sources, run `bazel run //:gazelle` to regenerate `BUILD.bazel` files — `gazelle_ci` will fail PRs otherwise. After bumping Go deps, run the `gazelle-update-repos` target if defined locally, or `bazel run //:gazelle -- update-repos -from_file=go.mod -to_macro=deps.bzl%go_dependencies -prune`.

## Architecture — the four phases

Gazelle runs in four ordered phases. Read `how-gazelle-works.md` for the full explanation; this is the orientation.

1. **Load** (`walk/`) — walk the directory tree, parse existing `BUILD`/`BUILD.bazel`, honor `# gazelle:exclude` and `.bazelignore`. Indexing modes: `all` (default, eager), `lazy` (visits only dirs requested via `GenerateResult.RelsToIndex`), `none`. Entry point: `walk.Walk2` → `walker.populateCache`.
2. **Generate** (`language/*`) — for each visited directory, every registered `language.Language` extension's `GenerateRules` produces target rules from sources. Extensions also emit `Imports` (opaque) for the resolver and may request lazy-index dirs.
3. **Resolve** (`resolve/`, extensions' `Resolver.Resolve`) — convert import strings to Bazel labels using the index built during Generate. Sets `deps`. Then `merger.MergeFile` reconciles new rules with what's already in the BUILD file.
4. **Write** (`merger/`, `rule/`) — format with `buildtools/build.Format` and write back. `-mode` flag controls write/diff/print.

### Extensions are everything

Anything language-specific is an extension. The Gazelle binary is statically linked: a `gazelle_binary` rule (see `def.bzl`) takes a list of `go_library` packages, each exporting `NewLanguage()` (or `New()`), and generates a `main` that registers them. Native extensions in this repo:

- `language/go/` — Go support; the largest extension and the historical reason Gazelle exists.
- `language/proto/` — `proto_library` rules. Language-specific proto rules (e.g. `go_proto_library`) live in `language/go/`.
- `language/bazel/` (and `language/base.go`, `lang.go`, `lifecycle.go`, `update.go`) — language-agnostic plumbing.
- `internal/language/test_filegroup` — test-only.

External extensions (Java, Python, Rust, JS/TS, Kotlin, Swift, C/C++, Haskell, R, Starlark) live in their own repos — see the README's "Supported languages" table.

### Top-level packages and what they own

| Pkg | Role |
|---|---|
| `cmd/gazelle/` | CLI entrypoint (`fix`, `update`, `update-repos` subcommands) |
| `cmd/autogazelle/`, `cmd/fetch_repo/`, `cmd/generate_repo_config/`, `cmd/move_labels/` | Auxiliary CLIs invoked by rules or workflows |
| `config/` | `Configurer` interface + global config; extensions extend this |
| `flag/` | Custom flag types shared across extensions |
| `label/` | Bazel label parsing/formatting |
| `language/` | The `Language` interface + base implementations |
| `merger/` | Merging generated rules with existing BUILD files; `# keep` semantics |
| `pathtools/` | Path manipulation helpers |
| `repo/` | External repo metadata (`go_repository`, `update-repos`) |
| `resolve/` | Import-to-label resolver + index types |
| `rule/` | Build-file AST wrappers (sits on top of `buildtools/build`) |
| `walk/` | Directory traversal + load-stage caching |
| `internal/` | Repository rules (`go_repository.bzl`, `go_repository_cache.bzl`, `go_repository_tools.bzl`, `is_bazel_module.bzl`, `overlay_repository.bzl`), bzlmod (`internal/bzlmod/`), version probing |
| `internal/bzlmod/` | `go_deps` module extension (the bzlmod equivalent of `update-repos`) |
| `tests/` | End-to-end and integration tests, including `tests/bcr/` BCR consumer modules |
| `v2/` | In-progress v2 of public packages (incremental migration; see commit `a472876`) |

### Public Starlark surface

- `def.bzl` — `gazelle`, `gazelle_binary` rules. Re-exports from `internal/`.
- `deps.bzl` — `gazelle_dependencies`, `go_repository` (WORKSPACE users).
- `extensions.bzl` — `go_deps` module extension (Bzlmod users).
- `go_tools.bzl` — exposes the embedded Go toolchain config.

`MODULE.bazel` declares deps on `rules_go`, `bazel_skylib`, `protobuf`, `rules_cc`, `rules_license`, `rules_shell`, `package_metadata`, `bazel_features`. Dev-only deps include `bazel_skylib_gazelle_plugin` and `stardoc`. The `go_deps` extension reads `go.work` to pin Go module deps.

## Editing rules and conventions

- **`# gazelle:` directives in BUILD files configure Gazelle** — they apply to the directory and all subdirs. Repo-root directives (see top `BUILD.bazel`) include `prefix github.com/bazelbuild/bazel-gazelle`, several `exclude` paths (`vendor`, `third_party`, `.github`, `internal/module/testdata`), and `go_naming_convention import_alias`.
- **`# keep` comments** in BUILD files prevent Gazelle from modifying that rule/attribute/element. Honor them.
- **Don't hand-edit generated BUILD files** for Go sources in this repo — re-run `bazel run //:gazelle`. The `gazelle_ci` target diffs and fails if you forgot.
- The repo dogfoods itself: the `gazelle_local` `gazelle_binary` (built from this checkout) is what `gazelle_ci` runs against the repo.

## Testing patterns

- Integration tests under `tests/` use `testtools` to set up fixture trees, run Gazelle, and assert generated BUILD output. Many subdirs there are named after the scenario they cover (`go_search`, `fix_mode_strict`, `go_proto_*`, `package_rule_*`, etc. — see `tests/` for the full list).
- `cmd/gazelle/gazelle_test.go` is the broad CLI integration test.
- `internal/go_repository_test.go`, `internal/runner_test.go` cover repo rules.
- `language/go/...` has unit tests for Go-specific generation/resolution.
- BCR end-to-end: `tests/bcr/go_work` and `tests/bcr/go_mod` are standalone Bazel modules used as consumers in CI.

## Contribution flow (from CONTRIBUTING.md)

1. Open or comment on a GitHub issue first to discuss design.
2. Add tests under `tests/` (or the appropriate package) for new behavior.
3. `bazel test //...` and `bazel run //:gazelle_ci` must pass.
4. Update `README.rst` if user-visible behavior or directives change.
5. CLA required (individual or corporate); reviewers will prompt.

## Documentation map

- `README.md` (rendered from `README.rst`) — user-facing setup and usage overview.
- `how-gazelle-works.md` — internals, the four phases, merger behavior, `# keep`.
- `extend.md` — guide to writing a new language extension.
- `gazelle-reference.md` — directives and flags reference.
- `language/go/reference.md`, `language/proto/reference.md` — language-specific directive references.
- `reference.md` — `gazelle` and `gazelle_binary` Starlark rule reference.
- `extensions.md` — `go_deps` bzlmod module extension reference.
