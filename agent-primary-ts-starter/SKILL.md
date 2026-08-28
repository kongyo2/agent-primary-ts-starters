---
name: agent-primary-ts-starter
description: Bootstrap or harden a TypeScript npm project the agent-primary way - tsconfig.json (TypeScript 7, maximally strict, machine-verifiable), Prettier (diff- and line-number-stable formatting), and Oxlint (bug-focused linting with machine-readable output), wired together with npm scripts and CI gates. Use whenever a TS npm project is being created, configured, or reviewed; whenever a tsconfig.json is created, edited, reviewed, troubleshot, or even just discussed - default to this skill instead of free-recall whenever the answer involves a tsconfig field; and whenever formatting or linting is being added or tuned - even if the user never says "tsconfig", "prettier", "oxlint", "formatter", or "linter".
---

# Agent-Primary TS + npm Starter (tsconfig · Prettier · Oxlint)

Use this when creating or configuring a TypeScript npm project, or when any of its three surfaces — `tsconfig.json`, Prettier, Oxlint — is touched individually. In a monorepo, edit the workspace the user names — ask if it isn't named.

## Design Premise

The reader, editor, and primary consumer of this project is an LLM agent, not a human scanning top-to-bottom. Every default below moves the project from human-oriented readability to machine verifiability and machine explorability.

- **Every mistake a flag can catch, a flag should catch.** Agents emit plausible-wrong code faster than anyone reviews it; the errors that type checks, tests, and lints raise unasked are the only feedback loop that scales. Structure the project so wrong code *snags* — on `tsc`, on the linter, on the test type-check — instead of sliding through to review.
- **The program must be legible file-by-file, without whole-program inference.** Inferred export signatures, ambient globals, and `tsc`-only path aliases are invisible to a grep, therefore invisible to the agent. Force them written down.
- **Tool output is a machine-consumed artifact.** Truncated types, ANSI escapes, and version-floating defaults are corruption in that channel — hence `tsc --pretty false` + `noErrorTruncation`, and `oxlint --format agent`.
- **Edits must not disturb what they don't touch.** Formatter defaults tuned for human eyes reflow whole regions around a one-line agent edit, shifting line numbers and invalidating the `old_string` matches the agent holds for its next edits. Formatting exists to keep diffs minimal and line numbers stable.
- **Prefer a checked element over a comment.** Inline and doc comments are not continuously maintained in AI-driven development — they rot silently. The same invariant as a type, `satisfies`, `assertNever`, or `@ts-expect-error` fails the build when it stops being true. See [Comments → Checked Artifacts](#comments--checked-artifacts).
- **A green `tsc` is not a working program.** `tsc` checks types, not module resolution at runtime. Config choices that are legal to the compiler and fatal to Node are this skill's most dangerous failure mode, so every emitting route ends in a runtime probe — see [Compile-Success Is Not Runtime-Success](references/tsconfig.md#compile-success-is-not-runtime-success).

This does **not** justify shrinking things for readability — no splitting files to keep them short, no avoiding long unions or deep generics, no simplifying types so a human can follow them linearly. Those optimize the false premise that a human reads top-to-bottom, at the cost of machine-verifiability.

## Applying to a Project

For a full bootstrap, apply in this order:

1. **tsconfig** — answer the [six questions](#before-any-tsconfig-decision), pick the [trio row](#the-target--module--moduleresolution-trio), copy [the strict core](#the-strict-core), then read [`references/tsconfig.md`](references/tsconfig.md) for the use-case template and gate configs. Read that file before creating or editing any tsconfig file.
2. **Prettier** — [section below](#prettier).
3. **Oxlint** — [section below](#oxlint).
4. **Scripts** — the [unified block](#npm-scripts).
5. **Verification** — [checks below](#verification), including the runtime probe from the reference.

For a single-surface task (just the linter, just a tsconfig question), jump straight to that section; the standing policy still applies.

## Before Any tsconfig Decision

Six answers decide the route. Ask for whatever isn't already evident; do not guess.

1. **Who emits JS** — `tsc`, a bundler, or a runtime that strips types?
2. **Who resolves the output** — Node, a browser via bundler, or another package's consumers? Independent of (1), and the usual source of broken configs.
3. **Lowest supported runtime** — the actual Node/browser floor, which fixes `target` and `lib`.
4. **JSX framework**, if any.
5. **Test runner**, and where test files live.
6. **Tools importing the TypeScript API** — typescript-eslint, ts-morph, Volar-based framework tooling. See [No Stable Compiler API in 7.0](references/tsconfig.md#no-stable-compiler-api-in-70).

Environments without a route in the reference — Deno, Workers, Bun-only, browser-without-bundler, mixed ESM/CJS packages — get the **standing policy** (the strict flags), not a template. Their `target`/`module`/`moduleResolution`/`lib` come from that runtime's own documentation.

## The `target` / `module` / `moduleResolution` Trio

Co-dependent. Pick one row, don't improvise. `target`/`lib` are floors to raise deliberately, not defaults.

| Scenario | `target` | `module` | `moduleResolution` | Template |
| --- | --- | --- | --- | --- |
| Bundled app | `es2024` | `preserve` | `bundler` | [§1](references/tsconfig.md#1-bundled-app-vite-nextjs-rspack-bun) |
| Node.js native ESM | `es2024` | `nodenext` | `nodenext` | [§2](references/tsconfig.md#2-nodejs-esm-application) |
| Library **emitted by tsc** | `es2022` | `nodenext` | `nodenext` | [§3](references/tsconfig.md#3-publishable-library-emitted-by-tsc) |
| Library **emitted by a bundler** | `es2022` | `preserve` | `bundler` | [§3](references/tsconfig.md#3-publishable-library-emitted-by-tsc) |
| CommonJS authored source | `es2022` | `commonjs` | `bundler` | [§6](references/tsconfig.md#6-commonjs) |

- **Never use `bundler` for output that `tsc` emits and Node loads.** `bundler` permits extensionless relative imports and `module: esnext`/`preserve` keeps them verbatim, so `export { x } from "./utils"` compiles clean and dies at `node dist/index.js` with `ERR_MODULE_NOT_FOUND`. `bundler` is correct only when a bundler resolves the result. This is the single most expensive mistake in this skill's subject area.
- `commonjs` + `bundler` is a **valid** combination as of 6.0 and the recommended landing spot off the removed `moduleResolution: node` — but see [§6](references/tsconfig.md#6-commonjs) for the strict-core exceptions it forces.
- `module: "preserve"` beats `"esnext"` for bundled apps: each import/export keeps its written form instead of being coerced.
- `nodenext` requires an extension on relative imports (TS2835). Either write `.js` yourself, or write `.ts`/`.tsx` and set `rewriteRelativeImportExtensions: true`, which rewrites those suffixes on emit. It does **not** add extensions to extensionless imports; there is no route that lets you omit them.

## The Strict Core

Identical in every route — copy it verbatim, then add the use-case block from the reference. [§6 CommonJS](references/tsconfig.md#6-commonjs) is the one documented exception.

```json
{
  "compilerOptions": {
    "skipLibCheck": true,
    "incremental": true,
    "moduleDetection": "force",
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
    "useDefineForClassFields": true,
    "resolveJsonModule": true,
    "allowJs": false,

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noPropertyAccessFromIndexSignature": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "allowUnreachableCode": false,
    "allowUnusedLabels": false,
    "noUncheckedSideEffectImports": true,

    "noErrorTruncation": true
  }
}
```

Per-flag rationale for the strictness flags — what each catches and what it costs — is in [Strict Flags Beyond `strict: true`](references/tsconfig.md#strict-flags-beyond-strict-true); `skipLibCheck` and `incremental` are loop-vs-gate concerns covered in [Two-Tier Checking](references/tsconfig.md#two-tier-checking).

### Standing Policy (override only with a stated reason)

1. **Correctness** — `strict: true` plus every flag in [Strict Flags Beyond `strict: true`](references/tsconfig.md#strict-flags-beyond-strict-true).
2. **Legibility without inference** — `isolatedDeclarations` (per route), `moduleDetection: "force"`, explicit `types: [...]` (never `["*"]`), `verbatimModuleSyntax` + `isolatedModules`, `erasableSyntaxOnly`.
3. **Output as a machine channel** — `noErrorTruncation: true`, and `--pretty false` plus a pinned `--checkers` on **every** agent-facing script, not just CI.
4. **Loop speed** — `skipLibCheck` and `incremental` on in the inner loop, off at the gate — except under `composite`, which forbids it. See [Two-Tier Checking](references/tsconfig.md#two-tier-checking).

Two documented exceptions, both in [§6 CommonJS](references/tsconfig.md#6-commonjs): `verbatimModuleSyntax` and `erasableSyntaxOnly` cannot both be on in a CJS-authored project. When the user proposes any other loosening, ask what concrete problem they are solving before agreeing.

Everything else about tsconfig — the TypeScript 7 baseline and its hard errors, per-route templates §1–§6, two-tier gate configs, monorepo project references, CommonJS, path aliases, determinism, migrating an existing tsconfig, and troubleshooting — is in [`references/tsconfig.md`](references/tsconfig.md). **Read it before writing any tsconfig file.**

## Prettier

Prettier's defaults are tuned for humans — its own docs recommend against `printWidth` over 80 "for readability". For an agent-primary project that premise inverts: wide reformat ripples around a one-line edit shift line numbers and silently invalidate the `old_string` matches and line references the agent is holding. These settings shrink that failure surface.

Install: `npm add -D prettier`. When prettier is already declared, keep its current version unless the user asks for a change.

Create `.prettierrc.json` if it does not exist:

```json
{
  "printWidth": 120,
  "trailingComma": "all",
  "arrowParens": "always",
  "semi": true,
  "endOfLine": "lf"
}
```

Why each value:

- **`printWidth: 120`** — the one deliberate deviation from the default (80). Wider width means a one-line agent edit rarely triggers auto-wrap of surrounding lines, so line counts stay stable across edits.
- **`trailingComma: "all"`** — appending an item to an array, object, or argument list is a single added-line diff instead of a two-line diff that also rewrites the previous line's `,`.
- **`arrowParens: "always"`** — adding a type annotation to an arrow parameter doesn't force inserting parens, so the annotation edit stays a localized change.
- **`semi: true`** — ASI quirks don't change behavior when an agent prepends a line starting with `[`, `(`, `+`, `-`, or `/`.
- **`endOfLine: "lf"`** — no whole-file CRLF reformat noise when Windows and Unix contributors share the repo.

`trailingComma`, `arrowParens`, `semi`, and `endOfLine` restate Prettier 3's current defaults — deliberately. Defaults move across majors (`trailingComma` flipped `es5`→`all` in 3.0, `endOfLine` flipped `auto`→`lf` in 2.0), and a pinned value keeps a `prettier` bump from silently reformatting the whole repo. Same doctrine as pinning `target`/`lib` in tsconfig.

Create `.prettierignore` if it does not exist (gitignore syntax; VCS directories and `node_modules` are ignored by default, and Prettier also follows `.gitignore` from the directory it runs in — the explicit list below survives repos where those assumptions fail):

```
node_modules
dist
build
coverage
*.min.js
package-lock.json
*.md
```

`*.md` is listed because Prettier's markdown formatter reflows paragraphs, list spacing, and table column widths across edits, which breaks `old_string` matches the same way reformatted TS would — and markdown is content, not code, so formatting consistency isn't worth the agent-edit cost.

When `.prettierrc.*` already exists, keep it and surface the diff against the values above; when `.prettierignore` exists, append only missing entries.

## Oxlint

The linter's consumer is the agent, so error-level findings are real bugs, style is the formatter's problem, and output is machine-parseable. Oxlint ships a dedicated `agent` value for `--format` — an output format the oxc team designed explicitly for LLM agent consumption — which the scripts below wire into the default `lint` command.

Install: `npm add -D oxlint`. When oxlint is already declared, keep its current version unless the user asks for a change.

Create `.oxlintrc.json` if it does not exist:

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["typescript", "unicorn", "oxc", "import", "promise", "node"],
  "categories": {
    "correctness": "error",
    "suspicious": "warn",
    "perf": "warn",
    "style": "off",
    "pedantic": "off"
  },
  "rules": {
    "no-console": "off"
  },
  "ignorePatterns": ["node_modules", "dist", "build", "coverage", "*.min.js"]
}
```

Why each value (verified against oxlint 1.80.0):

- **`$schema`** — the config itself becomes machine-verified: a typo'd key or wrong value type is flagged by any JSON-schema-aware editor or validator instead of being silently ignored.
- **`plugins`** — setting `plugins` **overwrites** the default plugin set (`typescript`, `unicorn`, `oxc`; ESLint core rules always stay on) rather than extending it, so the array restates the defaults, then adds `import`, `promise`, `node` for the TS/Node project surface. A list that omits `unicorn` or `oxc` silently drops their correctness rules — and with `style`/`pedantic` off, keeping them costs no noise, because categories gate severity across all plugins.
- **`categories.correctness: "error"`** — code that is definitely wrong blocks, so the agent fixes it before moving on. **`suspicious`/`perf: "warn"`** — informative without halting; the agent decides. **`style`/`pedantic: "off"`** — stylistic warnings in an agent edit loop drown out real bugs; style is Prettier's domain.
- **`rules.no-console: "off"`** — agent debugging routinely inserts `console.log`; flagging it slows the inner loop. Strip at release time with a separate gate, not on every lint.

This config is the untyped inner loop. At the gate, `oxlint --type-aware --type-check` (with `oxlint-tsgolint` installed) adds `typescript/no-floating-promises` and the `no-unsafe-*` family, which untyped lint and `tsc` both miss — see [The Gate](references/tsconfig.md#the-gate) for the setup and its constraints.

When `.oxlintrc.json` already exists, keep it. Surface the diff against the values above so the user can decide whether to migrate, but do not overwrite.

## npm Scripts

One block wires all three surfaces. Add to `package.json` (single-project; monorepos use the build-mode scripts in [§4](references/tsconfig.md#4-monorepo-with-project-references) instead):

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit --pretty false --checkers 4",
    "typecheck:watch": "tsc --noEmit --watch --pretty false --checkers 4",
    "typecheck:test": "tsc -p tsconfig.test.json --pretty false --checkers 4",
    "typecheck:test:ci": "tsc -p tsconfig.test.json --skipLibCheck false --incremental false --pretty false --checkers 4",
    "typecheck:ci": "tsc -p tsconfig.ci.json --pretty false --checkers 4",
    "check:decl": "tsc -p tsconfig.declarations.json --pretty false --checkers 4",
    "lint": "oxlint --format agent",
    "lint:fix": "oxlint --fix",
    "lint:strict": "oxlint --deny-warnings",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

- The same `--pretty false` and `--checkers N` on every `tsc` script: an agent reads all of them, so a script that keeps ANSI output or floats its checker count is an inconsistency the agent has to absorb.
- `--noEmit` is only meaningful in `typecheck` for a route whose base doesn't already set it; for [§2](references/tsconfig.md#2-nodejs-esm-application)/[§3](references/tsconfig.md#3-publishable-library-emitted-by-tsc) it overrides an emitting config, which is what you want in the loop.
- `check:decl` exists only where `tsconfig.declarations.json` does (§1/§2); pointing it at a §3 project is a TS5058 missing-file error — there the emitting build is its own declaration gate, so drop the script.
- `lint:fix` applies auto-fixable rule corrections in one pass — oxlint's `--fix` is the safe tier that doesn't change behavior.
- `lint:strict` promotes warnings to a non-zero exit code for CI gating.
- **Collision policy:** when `typecheck`, `lint`, or `format` already targets a different tool, leave it untouched and add `typecheck:tsc`, `lint:oxlint`, or `format:prettier` alongside.

## Existing Projects

Do not overwrite an existing `tsconfig.json`, `.prettierrc.*`, or `.oxlintrc.json`. Surface the diff against the matching template, flag standing-policy violations, and let the user decide. For tsconfig, follow the stepwise [Merge Rules](references/tsconfig.md#merge-rules-for-an-existing-tsconfig) — several strict flags produce hundreds of errors on first enable and are sequenced deliberately.

## Comments → Checked Artifacts

Inline and doc comments are not continuously maintained in AI-driven development — they rot silently while the code moves. When you find the left column, replace it with the right column: the same information, expressed in an element the toolchain maintains for you.

| Rotting comment | Checked replacement |
| --- | --- |
| `// @ts-ignore` | `// @ts-expect-error <reason>` — errors once fixed, so it self-deletes |
| `// this is a user id, not a name` | branded type: `string & { readonly brand: unique symbol }` |
| `// one of: 'a' \| 'b' \| 'c'` | literal union, or `as const` object + `(typeof X)[keyof typeof X]` |
| `// handle new variants here too` | `default: return assertNever(x)` |
| `// keep in sync with X` | derive from the one source of truth: `keyof`, `typeof`, `Extract`, mapped types |
| `// keep in sync with the config` | `satisfies Config` on the literal |
| `// this cast is safe because …` | a validator that actually checks, plus tests — a hand-written `x is T` predicate can still lie, so prefer an inferred predicate or a runtime check over asserting one |
| `// returns null if not found` | put it in the return type: `T \| null` |
| `// this function's shape is …` | `isolatedDeclarations` makes the signature mandatory |
| `// subtle overload, don't break it` | type-level test: `expectTypeOf` (vitest) or `tsd` in `*.test-d.ts` |

Keep two comment forms, because they are machine-consumed: `@deprecated` and `@ts-expect-error`. For sweeping the remaining unmaintained comments out of a codebase wholesale, the companion `ts-comment-purge` skill does it mechanically.

## Verification

- `npx tsc --showConfig -p <each config>` matches the table in [Two-Tier Checking](references/tsconfig.md#two-tier-checking) — check resolved values per file, not the snippets.
- `npm run typecheck`, `typecheck:test`, `typecheck:test:ci`, `typecheck:ci` (with no `.tsbuildinfo` present), and — §1/§2 — `check:decl` all exit 0.
- `npm run lint` exits 0 on a clean repository; `.oxlintrc.json` and `.prettierrc.json` parse as valid JSON.
- `npm run format` runs without error across the repository; `npm run format:check` exits 0 after a one-time `npm run format`.
- The emitted program actually runs — the runtime probe for the chosen route in [Compile-Success Is Not Runtime-Success](references/tsconfig.md#compile-success-is-not-runtime-success). A green `tsc` plus a failed `node dist/index.js` is a resolver-level config gap, not a code bug — diagnose against that section's cause list.
- A deliberate `const x: string = arr[0]` errors — confirms `noUncheckedIndexedAccess` is live.
- An error containing a long type shows it in full, not `... N more ...` — confirms `noErrorTruncation`.

Route-specific checks (monorepo gate traversal, declaration maps, tarball probes) are in the reference's [Verification](references/tsconfig.md#verification).

## References

- tsconfig deep reference (this skill): [`references/tsconfig.md`](references/tsconfig.md)
- TypeScript compiler options: <https://www.typescriptlang.org/tsconfig>
- Prettier options: <https://prettier.io/docs/en/options>
- Oxlint config: <https://oxc.rs/docs/guide/usage/linter/config>
- Oxlint CLI: <https://oxc.rs/docs/guide/usage/linter/cli>
- Oxlint type-aware linting: <https://oxc.rs/docs/guide/usage/linter/type-aware>
