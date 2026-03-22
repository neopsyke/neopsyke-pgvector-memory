# AGENTS

## Purpose
- Use this file for execution and contribution rules.
- Use `README.md` for product and runtime documentation.

## Working rules:
- Keep changes focused and minimal.
- Do not make unrelated refactors.
- Do not commit secrets, API keys, or local machine paths.
- Prefer ASCII in docs/code unless the file already uses Unicode.
- Preserve existing behavior unless the user asked for behavior changes.
- When fixing tests, always prioritize understanding the feature and making sure the root
  cause is addressed instead of making the test just pass.
- If you find a failing test, flaky test, even if unrelated to current changes
  make sure to find the root cause and fix it. Every work session must end with all
  tests running and stable.

## Gradle Execution Policy
- Do not run multiple Gradle commands in parallel from the same checkout of this repository.
- This project uses shared Gradle/Kotlin state under `.gradle/`, `.kotlin/`, and `build/`. Concurrent Gradle runs in one worktree can corrupt or contend on those caches and produce false failures.
- If you need multiple tasks, prefer a single Gradle invocation such as `./gradlew --no-daemon clean test fatJar releaseBundleZip`.
- If commands must run in parallel, each one must use an isolated checkout or worktree with its own project directory and build state.
- Do not assume separate terminals are enough. Parallel runs are only considered safe when the underlying checkout is different.
- If a concurrent Gradle failure appears, stop all daemons with `./gradlew --stop` and rerun the commands sequentially or from isolated copies.

