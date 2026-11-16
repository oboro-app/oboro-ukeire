---
name: testing
description: Use when writing or changing code, adding or fixing tests, debugging a failing test, or verifying behavior in the real app.
---

# Testing

Tests are evidence of completion. When behavior changes, make its test fail first, then fix. Acceptance criteria come from `bdd` skill scenarios.

## What a test suite must cover

Judge every change's tests on four axes, in this order.

| Axis | Rule |
|---|---|
| Scenarios first | Unit tests derive from the `bdd` scenarios. The user's real flows tell which inputs can happen: narrow impossible ranges with types instead of testing them, and decide from the scenario which error each failure returns |
| Boundaries and classes | Every unit test covers all boundary values and each equivalence class of its inputs (empty, one, many, limit, limit plus one, absent, invalid) that the types still allow |
| Density | Write many cases compactly (a table of named cases looped in one Rust test, or the stack's parameterized form). One readable table per behavior beats many copied tests |
| Coverage | Changed code is covered. Uncovered branches either get a case or a written reason |

Performance belongs in the verification items too: when a scenario or change touches a hot path (scanning source trees, ingesting large result files, aggregating records), add a measured check under equal load.

## Where to verify

The closer to the outside, the stronger the evidence and the slower. Pick the innermost layer that can catch what you want to verify.

| Layer | Target | Location |
|---|---|---|
| Module | ID parsing, state transitions, time handling, config | `#[cfg(test)]` in `crates/*/src/` |
| Cross-crate behavior | CLI output contract, schema files | `crates/ukeire-cli/tests/` (drives the real binary or library entry) |
| Harness | Check tooling | `scripts/test_*.py` |
| Real CLI | Look, exit codes, output a user reads | `cargo run -p ukeire-cli -- <subcommand>` on a fixture repo (steps below) |

## How to write

- Name the test after the behavior to protect. Do not repeat the function name.
- Use real things over mocks: a temporary directory tree as the target repo, real result files (JUnit XML, cucumber JSON, Playwright JSON, jsonl) for ingest, real config text for `ukeire.toml` handling.
- Put temporary files and directories in `tempfile` and always clean up, including child processes.
- Do not wait with `sleep`; wait with a deadline until a condition holds. The condition is "usable".
- When a fixture asserts a path or an ID, build it from the repo's own fixtures (for example `crates/ukeire-ingest/tests/fixtures/`) and keep the fixture in sync with the format definition it exercises.
- When adding a language definition or an ingest format, give it: a minimal positive case, a minimal negative case, and a case at the threshold (a boundary the rules decide differently). One table per definition beats scattered tests.
- When a report carries a location (file, line) or a stage and state, assert the exact value on a fixture; do not assert only that something was reported.
- When a test fails only occasionally, treat it as a bug that was passing by environmental luck. "Cannot reproduce" is not completion.

## New worktree

A worktree has an empty `target/`. The first `cargo test` or `clippy` builds every dependency and can far exceed 5 minutes; start that first build in the background (or with a raised timeout through `resources run`) rather than reading the timeout as a failure.

Run every cargo command through `resources run`, never bare, so the machine budget applies.

## When a test fails

For a statically decidable runtime bug, follow the [static-analysis skill](../static-analysis/SKILL.md).

1. Read the failure and identify the cause. Do not fix by guessing.
2. Decide whether code or test is right, using spec or the real thing.
3. If the test was wrong, fix it in a way whose reason you can explain.
4. Do not weaken expectations to pass. Do not delete tests. Do not clean up with `git checkout -- .` or `git reset --hard`.

A test that passes locally but fails in another checkout (different OS, parallel build load, file system differences) is an environment-dependent bug: repeat under the failing conditions before claiming a fix, and record conditions and pass/fail counts.

## Verifying with the real CLI

Even if automated tests pass, do not treat user-visible output as verified until produced by the real command on a real corpus.

1. Run `cargo run -p ukeire-cli -- <subcommand>` (the ones touched: `scan`, `ingest`, `manual`, `check`, `status`, `export`) on the repository itself or on a fixture repo (a temporary repo with a scenario source and result files).
2. Read the actual output: the stage and state of each item, counts, exit code. A command that exits 0 with wrong output is not verified.
3. For `--json`, validate the output against `docs/schema/` and the schema tests in `crates/ukeire-cli/tests/`.
4. If cleanup is needed (fixture repos, record files), verify the cleanup steps too, and report what verification left behind.

## Check scope

Per PR, build only what the change needs. This binds the implementer and every reviewer. Reuse recorded results for the same commit and command. Review the PR's check scope and run missing affected checks; do not repeat a check on each review round.

| Change | Run |
|---|---|
| Any | `cargo fmt --all --check` and `cargo clippy --workspace --all-targets -- -D warnings` |
| Touches Rust (`crates/`, `*.rs`, `Cargo.toml`) | Also `cargo test --workspace` with a filter for the changed scope when the full suite is not needed. No unfiltered run just to watch it pass |
| Touches docs or harness only | No cargo command beyond fmt/clippy on unchanged Rust (they pass unchanged). Run the scripts touched with `bash -n` (shell) or `ast.parse` (python) when `scripts/` exists |
| Full gate (fmt + clippy + test, the `[ci]` command in `.oboro/config.toml`) | oboro runs it at merge; also run it when the change touches the gate itself or the release |

`cargo test` takes one filter; it is a substring, not a regex. Run several tests with a shared substring, or several `cargo test` calls in one `resources run`. Write the scope run and not run in the PR's "Test notes".

A queued `resources run` reads the worktree when the run starts, not when it was queued. To keep fail-first evidence, commit only the failing tests and queue their run before writing any implementation.

## Reading check results

The gate chains its checks and stops midway on failure; later checks do not run. Do not filter the output; confirm the exit code.

## Report

Separate the check results (fmt, clippy, the filtered tests), the scope verified, and what was not verified. Do not mix passing tests with what was seen in the real CLI. A confirmation whose precondition was off does not count as verified.
