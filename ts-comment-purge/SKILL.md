---
name: ts-comment-purge
description: Unconditionally strip every line and block comment from a TypeScript codebase with npx @kongyo2/ts-comment-scanner - AST-based, so strings, template literals, regexes, and JSX text are never touched, and compiler/linter directives (@ts-expect-error, eslint-disable, oxlint-disable, prettier-ignore, ...) plus license headers are preserved automatically. Use whenever comments should be removed, stripped, or cleaned from TS/TSX code - purging AI-generated noise comments after an agent session (--diff scopes to changed files), enforcing a no-comment policy, or any "remove the comments" request, even when the user doesn't name the tool. Also covers scanning and inventorying comments (text/JSON/GitHub-annotation output) without removing them.
---

# TS Comment Purge (`@kongyo2/ts-comment-scanner`)

Use this to remove comments from TypeScript code, or to scan and report them. The tool is `npx @kongyo2/ts-comment-scanner` (Node ≥ 20; add `-y` under automation, or `npm add -D @kongyo2/ts-comment-scanner` to pin a version in the project).

## Policy

Inline and doc comments are not continuously maintained in AI-driven development — they rot silently while the code moves, so the standing policy is **zero non-directive comments**. Removal is unconditional: do not triage comments, do not ask about individual ones, and do not hand-preserve comments that look useful. Information worth keeping belongs in a checked element — a type, `satisfies`, `assertNever`, a test — per the Comments → Checked Artifacts table in the companion `agent-primary-ts-starter` skill; everything else is noise to delete mechanically. The only exceptions are the tool's protected classes below, which exist because deleting them breaks builds, lints, or license obligations.

Run the removal directly — this skill's policy is no per-comment confirmation, and the tool's own safety model backs that up: deletion follows the AST's comment ranges (strings, template literals, regexes, and JSX text are never touched), comment-only lines are removed whole, a trailing comment takes its leading whitespace with it, a space is inserted where block-comment removal would join tokens (`a/* x */b` → `a b`), writes are atomic (temp file + rename), UTF-8/UTF-16 encodings and BOMs are preserved, and after removal the tool re-scans each file and **errors without modifying it** if the result doesn't match expectations. Files with invalid encoding are reported and left unchanged. (`--dry-run` exists for a no-write preview when the user asks for one.)

## Removing Comments

```bash
# Whole project (node_modules and .git are excluded automatically)
npx -y @kongyo2/ts-comment-scanner --remove src

# Only files touched by uncommitted changes — cleanup after an agent session
npx -y @kongyo2/ts-comment-scanner --remove --diff HEAD

# Only files changed on the current branch
npx -y @kongyo2/ts-comment-scanner --remove --diff main...HEAD
```

Omitting paths scans the current directory. Default extensions are `.ts,.tsx,.mts,.cts` (`--ext` changes the set). The output reports `removed N (kept M)` per file — relay those counts.

`--diff <range>` takes anything `git diff` accepts as a revision: a single revision compares against the working tree (untracked files included, `.gitignore` respected); `a..b` / `a...b` compares commits. Unlike `--ignore`, `--diff` also filters explicitly listed files.

Only when the user explicitly asks to also remove directives or license headers:

```bash
npx -y @kongyo2/ts-comment-scanner --remove --remove-directives src   # directives too — breaks @ts-expect-error/eslint-disable dependents
npx -y @kongyo2/ts-comment-scanner --remove --remove-legal src        # license headers too — check license obligations first
```

## What Survives — and Must Stay

After a default `--remove`, anything still present was kept deliberately. **Never delete the survivors by hand**; the tool kept each one for a reason that hand-editing would break:

- **Directives** — comments that tools parse: compiler (`@ts-expect-error`, `@ts-nocheck`, `/// <reference>`), linters (`eslint-disable*`, `oxlint-disable*`, `biome-ignore`, …), formatters (`prettier-ignore`, `@format` pragmas), coverage (`istanbul ignore`, `c8 ignore`, `v8 ignore`), bundlers (`webpackChunkName:`, `@vite-ignore`, `#__PURE__`, `//# sourceMappingURL=`), test environments (`@vitest-environment`), security scanners (`NOSONAR`, `nosemgrep`, `codeql[...]`), spell checkers, editor regions. Detection matches each tool's real parser, and directives are reported tagged, e.g. `[@ts-expect-error]`.
- **Legal comments** — `/*! … */`, `@license` / `@preserve` / `@copyright`, and `SPDX-License-Identifier:` / `SPDX-FileCopyrightText:` tags. Removing license headers can violate the license; only `--remove-legal` on an explicit user request removes them.
- **Structurally protected comments** — resolved by *keeping* the comment, and no flag disables this. Two cases: a comment directly below a next-line directive (`@ts-expect-error`, `eslint-disable-next-line`) whose removal would delete that line and shift the directive's target; and a comment whose removal would newly activate a later position-dependent pragma (Prettier's `@format`/`@prettier`, valid only as the file's first comment; Bun's `// @bun`, valid only at the top of the file).

## Scanning Without Removing

```bash
npx -y @kongyo2/ts-comment-scanner src                        # text report: file:line:col [kind] text
npx -y @kongyo2/ts-comment-scanner --json src                 # machine-readable; summary has files/comments/directives counts
                                                              # with --remove: summary.changedFiles = files actually modified,
                                                              # summary.files = files with a removal, keep, or skip
npx -y @kongyo2/ts-comment-scanner --skip-directives src      # non-directive comments only
npx -y @kongyo2/ts-comment-scanner --only-directives src      # directives only
npx -y @kongyo2/ts-comment-scanner --remove --dry-run src     # what --remove would delete, without writing
```

## CLI Reference

| Flag | Effect |
| --- | --- |
| `--format <text\|json\|github>` | Output format (`--json` is shorthand; `github` emits workflow-command annotations) |
| `--ignore <glob>` | Exclude files/directories (repeatable; a glob without `/` matches file names, with `/` matches paths; explicitly listed files are exempt) |
| `--ext <list>` | Comma-separated extensions to scan (default `.ts,.tsx,.mts,.cts`) |
| `--diff <range>` | Restrict to files git reports as changed |
| `--skip-directives` / `--only-directives` | Filter directives out of / down to the report |
| `--fail-on-comment` | Exit 1 when any comment is reported — the CI gate flag |
| `--remove` | Delete reported comments in place |
| `--dry-run` | With `--remove`: show what would be deleted, change nothing |
| `--remove-directives` / `--remove-legal` | With `--remove`: also delete directives / legal comments (structural protection has no off switch) |

Exit codes: `0` success · `1` `--fail-on-comment` found comments · `2` argument or runtime error.

Note for gating: `--skip-directives --fail-on-comment` still exits 1 on retained **legal** comments (verified) — in a repo with license headers, `--ignore` those files or accept the finding.

## Verification

- Re-run with `--remove --dry-run` on the same scope: it prints `No removable comments found` and exits 0 — the purge is complete and idempotent.
- The project's own gates stay green — typecheck, lint, tests (`npm run typecheck` / `lint` / `test`, or `npx tsc --noEmit --pretty false` when no scripts exist). Directives were preserved, so `@ts-expect-error` and `eslint-disable`/`oxlint-disable` comments still bind to the lines they governed.
- If the tool reported any file as errored (unexpected post-removal state, invalid encoding), that file was left unmodified — surface it to the user instead of editing it by hand.

## References

- npm: <https://www.npmjs.com/package/@kongyo2/ts-comment-scanner>
- GitHub: <https://github.com/kongyo2/ts-comment-scanner>
- Project site: <https://kongyo2.github.io/ts-comment-scanner/>
