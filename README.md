# bun-test-runner

[![npm](https://img.shields.io/npm/v/bun-test-runner.svg)](https://www.npmjs.com/package/bun-test-runner)
[![Bun](https://img.shields.io/badge/Bun-%3E%3D1.3.0-blue?logo=bun)](https://bun.sh)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

A CLI (`runtest`) for coding agents. It runs `bun test`, `tsc` and `biome` and prints short, precise, structured results.

- **Less context pollution:** no passed-test noise, no repeated stack frames, capped and compact lists.
- **Fewer tokens:** Antigravity (AGY) tends to ingest huge blocks of tool output (due to massive context window) while Claude Code tends to truncate output more aggressively (with `runtest` AGY ingested tool output up to 14x smaller).
- **Fewer agent calls:** gives the agent only most relevant output it needs (to verify or fix failures); fewer follow-up runs, file reads and raw tool calls.

Measure what agents run and receive: [agent-tool-call-tracer](https://github.com/martin-winkler-dev/agent-tool-call-tracer).
- Plugin for *Claude Code* and *Antigravity 2.0 (AGY)*.
- Logs every agent tool call into a local SQLite database.
- Compare before/after or similar projects - compare tool call density and ingested tool output length.

## Highlights

- **Ultra short output**

```txt
$ bun runtest test
STATUS: PASS
TOTAL_TESTS: 3 (2 files)
```

- **Combined audit for: typecheck, lint, tests**

```txt
$ bun runtest audit
STATUS: FAIL
TOTAL_TESTS: 3 (2 files)
TYPECHECK: 2 errors (src/user.ts)
LINT: 2 errors (src/compare.ts, src/messy.ts)
TEST: 2 failed (tests/math.test.ts, tests/user.test.ts)
NEXT: If allowed to edit files, run: bun runtest check
```

- **Precise test report including test code**

```txt
$ bun runtest test
STATUS: FAIL
TOTAL_TESTS: 3 (2 files)
TOTAL_FAILED: 2
TEST_FILE: tests/math.test.ts:5:23
TEST_NAME: adds
ERROR: expect(received).toBe(expected)
EXPECTED: 3
RECEIVED: 2
TEST_BLOCK:
4: test('adds', () => {
5:     expect(add(1, 1)).toBe(3);
6: });
OTHER_FAILURES: 1 in 1 group
expect(received).toBe(expected) EXPECTED: "Hello Ada!" RECEIVED: "Hello Ada" (1)
  tests/user.test.ts:5:45 greets by name
```

## Installation

```sh
bun add -d bun-test-runner
```

- **CLI only:** installs the command `runtest`; no importable API.
- **Bun only:** requires [Bun](https://bun.sh) >=1.3.0; does not run on Node.js. `bun` must be on the `PATH`, also when installed with npm.
- **TypeScript:** `typecheck`, `check` and `audit` run `tsc` from `./node_modules/typescript`. runtest depends on `typescript`, so npm, yarn and bun's hoisted install usually place it there; pnpm and bun's isolated install do not. Add `typescript` as a dev dependency to be sure and to pin the version.
- **Biome:** `lint`, `check` and `audit` run `biome` from `./node_modules/@biomejs/biome`. Add `@biomejs/biome` as a dev dependency.

| Tool | Tested with |
| :-- | :-- |
| bun | 1.3.14, 1.4.2 |
| TypeScript | 6.0.3 |
| Biome | 2.5.14 |

Verify the installation:

```sh
bun runtest --version
```

## How to use

Copy the snippet below into the agent instructions (`AGENTS.md`, `CLAUDE.md`). Modify as needed.

```md
## Tests, typecheck and lint
Run `bun runtest <command> [paths] [options]` from the project root (command first); it wraps `bun test`, `tsc` and `biome` and prints only compact results.

- `bun runtest check [paths]`: applies safe lint fixes (writes files), then typecheck, lint and tests; stops at the first failing stage and shows its first failure in detail, then the location and reason of its other failures. Run after code changes; fix and re-run until `STATUS: PASS`.
- `bun runtest audit [paths]`: the same checks as `check`, read-only; runs all stages, one summary line each.
- `bun runtest test [paths]`: failure count, the first failing test with its code, then every other failing test with location and reason.
- `bun runtest failures [paths]`: all failing tests by file with location and reason, no code.
- `bun runtest coverage [paths]`: uncovered lines per file.
- `bun runtest typecheck` (whole project, no paths), `bun runtest lint [paths]`: one error with code context, then every error grouped (`ERROR_GROUPS:`); `lint --fix` applies safe fixes.

Test options: `-t <name regex>`, `--timeout <ms>`, `--env-file <file>`; `-p <tsconfig>` for `typecheck`, `check` and `audit`. In `check` and `audit`, `[paths]` and test options only select tests; typecheck and lint always cover the whole project. Other flags are rejected.
Output starts with `STATUS: PASS|FAIL|ERROR`. Read all of it, never through `head`/`tail`, and do not redirect it to a file: the output is complete and capped. Follow a `NEXT:` hint only if its condition applies.
Fallback: run `bun test`, `tsc` or `biome` directly only when runtest output is not enough for the task (e.g. `STATUS: ERROR` without a clear cause, or a needed tool option runtest does not accept).
```

## Commands

Usage: `bun runtest [action] [options] [paths...]`, run from the project root. The action comes first; options and paths may follow in any order.

| Command | Description |
| :-- | :-- |
| `audit [paths]` (default) | Typecheck, lint (read-only) and tests; one summary line per stage |
| `check [paths]` | Applies safe lint fixes, then typecheck, lint and tests; stops at the first failing stage and shows it in detail |
| `test [paths]` | Tests; the first failure with its test code, then every other failure with location and reason |
| `failures [paths]` | All failing tests, grouped by file, with location and reason |
| `coverage [paths]` | Tests with line and function coverage; uncovered lines per file |
| `lint [paths]` | `biome check`; one error in detail, all errors grouped by rule and message |
| `typecheck` | `tsc --noEmit`, whole project; one error in detail, all errors grouped by code and message |
| `help`, `-h`, `--help` | Shows help |
| `version`, `-v`, `--version` | Shows the version |

| Option | Description | Used by |
| :-- | :-- | :-- |
| `-t, --test-name-pattern <regex>` | Filter tests by name | `test`, `failures`, `coverage`, `check`, `audit` |
| `--timeout <ms>` | Per-test timeout | `test`, `failures`, `coverage`, `check`, `audit` |
| `--env-file <file>` | Load environment variables from a file | `test`, `failures`, `coverage`, `check`, `audit` |
| `--fix` | Apply safe lint and formatting fixes | `lint` (`check` always fixes) |
| `-p, --project <path>` | Custom tsconfig | `typecheck`, `check`, `audit` |

- An option for a tool the command does not run is ignored (e.g. `-p` on `test`, `-t` on `lint`); `--fix` is rejected outside `lint` and `check`.
- Unknown and unsupported options (e.g. `-u`, `--watch`, `--bail`) give exit code 2; unsupported ones include a hint.
- In `check` and `audit`, paths and test options only select tests. Typecheck and lint always cover the whole project.

### Output

- **First line:** `STATUS: PASS`, `STATUS: FAIL` or `STATUS: ERROR`. `check` adds the failing stage, e.g. `STATUS: FAIL (lint)`; on pass, `check` and `audit` list the stages that ran, e.g. `STATUS: PASS (typecheck, lint, test)`.
- **Details:** `KEY: value` lines, e.g. `TOTAL_TESTS:`, `TOTAL_FAILED:`, `FILE:`, `ERROR:`, `CODE_BLOCK:`.
- **`TOTAL_TESTS:`:** also on `PASS`, so a path or `-t` filter visibly matched tests.
- **`NEXT:`:** a hint for a follow-up command, printed only in a few cases.
- **Exit codes:** `0` pass, `1` failures, `2` error (invalid option, missing tool, tool crash).
- **Size:** lists and code blocks are capped; the whole output is limited to 300 lines and 1000 characters per line.
- **Examples:** see [Highlights](#highlights).

### Usage log (optional)

runtest records every run in a local SQLite file when `.smartin/test-runner/log.sqlite` exists in the project root. An empty file turns it on; deleting it turns it off.

```sh
mkdir -p .smartin/test-runner && touch .smartin/test-runner/log.sqlite
```

- **`runs`:** one row per call: arguments, exit code, duration, full output and parsed fields (`status`, `total_failed`, ...).
- **`commands`:** one row per tool call (`bun-test`, `tsc`, `biome`); raw stdout/stderr only when the tool failed.
- Logging never changes output or exit code.
- Add `/.smartin/` to `.gitignore`: the log can contain source excerpts and raw tool output.

## Limitations

- **Project root only:** must run in the folder that contains `package.json`; no search in parent folders.
- **Tools from `./node_modules`:** no global or parent-folder lookup; a missing `typescript` or `@biomejs/biome` gives `STATUS: ERROR`.
- **Three tools only:** bun's test runner, `tsc` and `biome`.
- **Few options:** only the options above; no pass-through (`--`).
- **No interactive modes:** no watch mode, no snapshot updates, no `--bail`.
- **One failure in detail:** other failures are listed with location and reason only (up to 60). Use `failures` for the full list.
- **Coverage:** lines and functions only, no branch coverage; only files the tests load.
- **Tool versions are not checked:** runtest reads bun's human-readable output, which may change between bun versions.
- **Writes files:** `check` and `lint --fix` apply Biome's safe fixes. Solution-style tsconfigs run `tsc -b`, which may write `.tsbuildinfo` files.
- **Test code lookup:** by call name (`test`, `it`, `describe`, hooks). Renamed imports or calls like `bun.test()` are not recognized.

## License

Copyright 2026 Martin Winkler

Licensed under the [Apache License 2.0](LICENSE).
