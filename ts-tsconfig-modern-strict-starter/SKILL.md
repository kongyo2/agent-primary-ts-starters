---
name: ts-tsconfig-modern-strict-starter
description: Set up a TypeScript 7 tsconfig.json with maximally strict, machine-verifiable settings tuned so the LLM agent doing the editing gets the largest possible share of its mistakes reported as compiler errors, in output it can consume. Use whenever a tsconfig.json is being created, edited, reviewed, troubleshot, or even just discussed — apply the standing policy even when the user only asks about a single option. Default to this skill instead of free-recall whenever the answer involves a tsconfig field.
---

# TS + npm tsconfig — Maximum Strict, Agent-Verifiable

Use whenever a `tsconfig.json` is being created, edited, reviewed, or troubleshot in a TypeScript npm project. In a monorepo, edit the workspace the user names — ask if it isn't named.

## Design Premise

The reader is an LLM agent, not a human scanning top-to-bottom.

- **Every mistake a flag can catch, a flag should catch.** Agents emit plausible-wrong code faster than anyone reviews it; type errors are the only feedback that arrives unasked.
- **The program must be legible file-by-file, without whole-program inference.** Inferred export signatures, ambient globals, and `tsc`-only path aliases are invisible to a grep, therefore invisible to the agent. Force them written down.
- **Compiler output is a machine-consumed artifact.** Truncated types, ANSI escapes, and version-floating defaults are corruption in that channel.
- **Prefer a checked element over a comment.** A comment rots silently; the same invariant as a type, `satisfies`, `assertNever`, or `@ts-expect-error` fails the build. See [Comments → Checked Artifacts](#comments--checked-artifacts).
- **A green `tsc` is not a working program.** `tsc` checks types, not module resolution at runtime. Config choices that are legal to the compiler and fatal to Node are this skill's most dangerous failure mode, so every emitting route ends in a runtime probe. See [Compile-Success Is Not Runtime-Success](#compile-success-is-not-runtime-success).

This does **not** justify shrinking things for readability — no splitting files to keep them short, no avoiding long unions or deep generics. Those optimize linear human reading at the cost of machine-verifiability.

## Before Applying Anything

Six answers decide the route. Ask for whatever isn't already evident; do not guess.

1. **Who emits JS** — `tsc`, a bundler, or a runtime that strips types?
2. **Who resolves the output** — Node, a browser via bundler, or another package's consumers? Independent of (1), and the usual source of broken configs.
3. **Lowest supported runtime** — the actual Node/browser floor, which fixes `target` and `lib`.
4. **JSX framework**, if any.
5. **Test runner**, and where test files live.
6. **Tools importing the TypeScript API** — typescript-eslint, ts-morph, Volar-based framework tooling. See [No Stable Compiler API in 7.0](#no-stable-compiler-api-in-70).

Environments without a route below — Deno, Workers, Bun-only, browser-without-bundler, mixed ESM/CJS packages — get the **standing policy** (the strict flags), not a template. Their `target`/`module`/`moduleResolution`/`lib` come from that runtime's own documentation.

## TypeScript 7 Baseline

Assume TS 7.0+ (GA 2026-07-08, Go-based, ~8–12x faster than 6.0 on full builds; behaviors and error codes in this file re-verified against 7.0.2). From 5.x, migrate 5.x → 6.0 (warnings) → 7.0 (hard errors).

New defaults in 6.0/7.0: `strict: true`, `module: esnext`, floating `target`, `noUncheckedSideEffectImports: true`, `libReplacement: false`, `stableTypeOrdering: true` (immutable), `types: []` (no auto `@types/*`), and `rootDir: "./"` — no longer inferred; an emitting config whose sources sit under `src/` errors TS5011 until `rootDir` is written.

Hard errors in 7.0 — fix these before anything else works:

- `target: es5`, `downlevelIteration`
- `moduleResolution: node` / `node10` / `classic`
- `module: amd` / `umd` / `systemjs` / `none`
- `baseUrl`, `outFile`
- `esModuleInterop: false`, `allowSyntheticDefaultImports: false`, `alwaysStrict: false`
- `module` keyword for namespaces (TS1540); `assert` on imports (TS2880 — use `with`)

Downlevel emit reaches back to `es2015`, the 6.0 floor: async→`__awaiter` at `es2015` and standard decorators→`__esDecorate` both emit on 7.0.2, so a low `target` is no longer a reason to stay on 6.0.

**Explicitness rule.** Write an option out when its default is version-dependent, or when its absence is indistinguishable from deliberate loosening — `target`, `lib`, `module`, `moduleResolution`, `types`, `rootDir`, `strict`, and every strictness flag. For the rest, `tsc --showConfig` prints the merged `extends` result — 7.0 no longer expands defaults into it, so an option that matters must be written to be visible at all.

**Always pin `target` and `lib`.** The 7.0 default target floats with the compiler version, so a `typescript` bump silently changes emit and the ambient lib set. `lib` describes what the *runtime* has; `target` describes what syntax to emit. For a library, both come from the lowest supported consumer, not from your dev machine.

### No Stable Compiler API in 7.0

Anything that does `import * as ts from "typescript"` — typescript-eslint, ts-morph, `type-coverage`, custom transformers, some bundler plugins — does not run against 7.0: the `typescript@7` package ships no JS API, and a new API is planned for 7.1 but not shipped, so check each tool's own release notes rather than assuming a version. Volar-based tooling is affected too, so Vue, Svelte, Astro, and MDX projects stay on 6.0 for the language service, and Angular can use 7.0 for CLI `tsc` only. Run the two side by side via npm aliases:

```json
{
  "devDependencies": {
    "@typescript/native": "npm:typescript@^7.0.2",
    "typescript": "npm:@typescript/typescript6@^6.0.2"
  }
}
```

Check each tool before assuming it needs this — some vendor their own TypeScript (`attw` does) and are unaffected.

## Standing Policy (override only with a stated reason)

1. **Correctness** — `strict: true` plus every flag in [Strict Flags Beyond `strict`](#strict-flags-beyond-strict-true).
2. **Legibility without inference** — `isolatedDeclarations`, `moduleDetection: "force"`, explicit `types: [...]` (never `["*"]`), `verbatimModuleSyntax` + `isolatedModules`, `erasableSyntaxOnly`.
3. **Output as a machine channel** — `noErrorTruncation: true`, and `--pretty false` plus a pinned `--checkers` on **every** agent-facing script, not just CI.
4. **Loop speed** — `skipLibCheck` and `incremental` on in the inner loop, off at the gate — except under `composite`, which forbids it. See [Two-Tier Checking](#two-tier-checking).

Two documented exceptions, both in [§6 CommonJS](#6-commonjs): `verbatimModuleSyntax` and `erasableSyntaxOnly` cannot both be on in a CJS-authored project. When the user proposes any other loosening, ask what concrete problem they are solving before agreeing.

## Two-Tier Checking

The inner loop and the gate want opposite things; collapsing them yields either a slow loop or a weak gate.

| | `tsconfig.json` (loop) | `tsconfig.ci.json` (gate) | `tsconfig.declarations.json` (gate) | `tsconfig.test.json` |
| --- | --- | --- | --- | --- |
| `incremental` | `true` | `false` | `false` | inherit |
| `skipLibCheck` | `true` | `false` | inherit | inherit |
| `isolatedDeclarations` | — | — | `true` | — |
| Emit | per route | `noEmit: true` | `emitDeclarationOnly` → scratch | `noEmit: true` |

Every one of those values must be written **explicitly in the extending file**. `extends` merges; it does not reset. A gate that inherits `noEmit: false` from an emitting base will write `dist/` during CI, and a gate that inherits `incremental: true` will read the cache it was created to bypass. Verify with `tsc --showConfig -p tsconfig.ci.json`, not by reading the snippet.

- **`skipLibCheck`** — one broken `.d.ts` in `node_modules` stops the whole inner loop; never checking it means a dependency bump breaks your types with no signal. Check it once, at the gate.
- **`incremental`** — the cache keeps the loop near-instant. `tsc -b` does track `.d.ts` changes across project references, so a cache-free gate is defense-in-depth against restored VCS timestamps, hand-deleted outputs, and compiler bugs — not a fix for a routine unsoundness.
- **`isolatedDeclarations`** — needs `declaration` or `composite` (TS5069), awkward under `noEmit`. A separate declaration-only project sidesteps the interaction.
- **`composite`** — forbids `incremental: false` (TS6379). Monorepos use `tsc -b --force` instead; see [§4](#4-monorepo-with-project-references).

## Pick the Use Case First

```
Who emits the JS that ships?
├── A bundler or a type-stripping runtime ─────► §1  Bundled app
└── tsc
    ├── You run it (Node, ESM) ────────────────► §2  Node ESM app
    └── Someone else consumes it (npm)
        ├── Single package ────────────────────► §3  Library
        └── Monorepo ──────────────────────────► §4  Project references
Add to §1–§3 ──────────────────────────────────► §5  Gate configs
CommonJS anywhere ─────────────────────────────► §6  CommonJS
```

## The `target` / `module` / `moduleResolution` Trio

Co-dependent. Pick one row, don't improvise. `target`/`lib` are floors to raise deliberately, not defaults.

| Scenario | `target` | `module` | `moduleResolution` |
| --- | --- | --- | --- |
| Bundled app | `es2024` | `preserve` | `bundler` |
| Node.js native ESM | `es2024` | `nodenext` | `nodenext` |
| Library **emitted by tsc** | `es2022` | `nodenext` | `nodenext` |
| Library **emitted by a bundler** | `es2022` | `preserve` | `bundler` |
| CommonJS authored source | `es2022` | `commonjs` | `bundler` |

- **Never use `bundler` for output that `tsc` emits and Node loads.** `bundler` permits extensionless relative imports and `module: esnext`/`preserve` keeps them verbatim, so `export { x } from "./utils"` compiles clean and dies at `node dist/index.js` with `ERR_MODULE_NOT_FOUND`. `bundler` is correct only when a bundler resolves the result. This is the single most expensive mistake in this file's subject area.
- `commonjs` + `bundler` is a **valid** combination as of 6.0 and the recommended landing spot off the removed `moduleResolution: node` — but see [§6](#6-commonjs) for the strict-core exceptions it forces.
- `module: "preserve"` beats `"esnext"` for bundled apps: each import/export keeps its written form instead of being coerced.
- `nodenext` requires an extension on relative imports (TS2835). Either write `.js` yourself, or write `.ts`/`.tsx` and set `rewriteRelativeImportExtensions: true`, which rewrites those suffixes on emit. It does **not** add extensions to extensionless imports; there is no route that lets you omit them.

## The Strict Core

Identical in every template below — copy it verbatim, then add the use-case block. §6 is the one documented exception.

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

## §1. Bundled app (Vite, Next.js, Rspack, Bun)

```json
{
  "compilerOptions": {
    "target": "es2024",
    "module": "preserve",
    "moduleResolution": "bundler",
    "lib": ["es2024", "dom"],
    "jsx": "react-jsx",
    "types": [],
    "rootDir": "./src",
    "noEmit": true,
    "allowImportingTsExtensions": true
  },
  "include": ["src"],
  "exclude": ["dist", "build", "coverage"]
}
```

`dom.iterable` and `dom.asynciterable` are folded into `dom` as of 6.0 — listing them is a no-op. Drop `"dom"` for non-browser code. JSX varies by framework and the framework's own docs win: React and Preact both use `react-jsx` (Preact adds `"jsxImportSource": "preact"`), Solid uses `"jsx": "preserve"` with `"jsxImportSource": "solid-js"` and lets its Babel preset transform. `types: []` is right only if the app uses no ambient globals — otherwise list exactly what it needs (`["node"]`, `["vite/client"]`).

This template is React/DOM-shaped. Non-React, non-DOM, or framework-CLI-generated projects take the strict core and their own trio row.

## §2. Node.js ESM application

`package.json` must have `"type": "module"`. Relative imports need `.js`. The `es2024` row assumes a Node ≥ 22 floor — `lib: ["es2024"]` type-checks `Object.groupBy` and `Promise.withResolvers`, which Node 18/20 lack at runtime, so an older supported floor lowers `target`/`lib` per question 3 of [Before Applying Anything](#before-applying-anything).

```json
{
  "compilerOptions": {
    "target": "es2024",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["es2024"],
    "types": ["node"],
    "rootDir": "./src",
    "outDir": "./dist",
    "sourceMap": true,
    "noEmitOnError": true
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts", "dist"]
}
```

`noEmitOnError` matters more for an agent than a human: a human sees red text before running the output, an agent chains straight into `node dist/…` and debugs the wrong layer. The `exclude` above keeps tests out of the shipped build, which means they are unchecked until you add [`tsconfig.test.json`](#5-gate-configs) — do that in the same change, or the strictest config in the repo is silently skipping a third of it.

## §3. Publishable library (emitted by tsc)

Self-contained — do not derive it from §2, whose `target` and `types` are wrong for a library.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["es2022"],
    "types": [],
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "isolatedDeclarations": true,
    "noEmitOnError": true,
    "allowImportingTsExtensions": false
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts", "dist"]
}
```

`target`/`lib` are the **lowest consumer runtime you support**, not your dev version. Internal relative imports must be written `./thing.js`; `nodenext` enforces this, which is exactly why it's here and `bundler` is not. A library emits declarations anyway, so `isolatedDeclarations` lives in the main config and no separate §5 declaration project is needed.

Configure `package.json` `exports` with `types` first in each conditional block, then verify it mechanically — see [The Gate](#the-gate). If a bundler produces your dist instead of `tsc`, use the bundler row from the trio table and test the bundled output against a real consumer.

## §4. Monorepo with project references

Root `tsconfig.json` — `"files": []` keeps the root from compiling sources directly, which would double-compile everything the leaves already build:

```json
{
  "files": [],
  "references": [{ "path": "./packages/core" }, { "path": "./packages/app" }]
}
```

`tsconfig.base.json` holds the strict core and uses `${configDir}` so paths resolve against the *extending* config:

```json
{
  "compilerOptions": { "rootDir": "${configDir}/src", "outDir": "${configDir}/dist" }
}
```

Each package extends the base **and its use-case row** (§1, §2, or §3 — a leaf still has to decide its own `target`/`lib`/`module`/`types`):

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "isolatedDeclarations": true
  },
  "include": ["src/**/*"],
  "references": [{ "path": "../core" }]
}
```

**Monorepo gates are not §5.** Two constraints make the single-project gates wrong here:

- `tsc -p` on a solution root does **not** traverse `references`. Run it against the root above and it checks nothing — a broken leaf still exits 0. Only `tsc -b` walks the graph.
- `composite: true` forbids `incremental: false` (TS6379), so `tsconfig.ci.json` cannot be inherited by a leaf.

So a monorepo uses build mode throughout, with `--force` standing in for the cache-free gate:

```json
{
  "scripts": {
    "typecheck": "tsc -b --pretty false",
    "typecheck:ci": "tsc -b --force --pretty false --builders 4",
    "check:libs": "tsc -p tsconfig.libcheck.json --pretty false --checkers 4",
    "clean": "tsc -b --clean"
  }
}
```

`tsconfig.libcheck.json` is a non-composite, no-emit project that exists only to run the dependency-types sweep that `composite` blocks:

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": { "noEmit": true, "incremental": false, "skipLibCheck": false, "rootDir": "." },
  "include": ["packages/*/src/**/*"]
}
```

The `rootDir` override is load-bearing: `${configDir}` resolves against the *extending* config, so the inherited `rootDir` lands on `<root>/src` and every `packages/*` file errors TS6059 without it.

Leaves emit their own declarations under `isolatedDeclarations`, so `tsc -b --force` is also the declaration gate.

## §5. Gate configs

For §1–§3. Every value is restated rather than inherited, because `extends` merges.

```json
// tsconfig.ci.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "noEmit": true, "incremental": false, "skipLibCheck": false }
}
```

```json
// tsconfig.declarations.json  — §1 and §2 only; §3 already emits declarations
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": false,
    "incremental": false,
    "declaration": true,
    "emitDeclarationOnly": true,
    "isolatedDeclarations": true,
    "outDir": "./node_modules/.cache/decl"
  }
}
```

Don't override `allowImportingTsExtensions` here: `emitDeclarationOnly` permits it, and forcing it off breaks a §1 project that imports with `.ts` extensions.

```json
// tsconfig.test.json  — required wherever the main config excludes tests
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "noEmit": true, "rootDir": "./", "types": ["node", "vitest/globals"] },
  "include": ["src/**/*", "test/**/*"],
  "exclude": ["dist"]
}
```

Set `types` to your runner's globals package. Three inherited values must be overridden, not assumed: `include` in an extending config replaces the base's (restate the source globs), while `exclude` and `rootDir` merge through — left alone, the base's `**/*.test.ts` exclude silently drops every test file from the program, and files under `test/` error TS6059 for sitting outside the inherited `rootDir: "./src"` (both verified on 7.0.2, including under `noEmit`).

## §6. CommonJS

The only route that breaks the strict core, so it needs its own recipe rather than a table row.

`module: "commonjs"` + `moduleResolution: "bundler"` is valid, but `verbatimModuleSyntax: true` and `erasableSyntaxOnly: true` are **mutually exclusive** in CJS-authored source: verbatim mode requires `import x = require(…)` / `export =`, and `erasableSyntaxOnly` bans exactly that syntax. Writing ESM `export` instead gives `TS1287`; writing `export =` gives `TS1294`. Pick one:

- **Author ESM, ship CJS (preferred).** Keep §2 or §3 unchanged for type-checking and let a bundler (tsdown, rolldown, tsup) produce the CJS artifact. The strict core stays intact.
- **Author CJS with `import =` syntax.** Keep `verbatimModuleSyntax: true`, set `"erasableSyntaxOnly": false`, and record why in the config.
- **Author CJS with ESM syntax and let tsc downlevel.** Keep `erasableSyntaxOnly: true`, set `"verbatimModuleSyntax": false`, and record why.

## Strict Flags Beyond `strict: true`

`strict` covers `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `strictBuiltinIteratorReturn`, `noImplicitThis`, `useUnknownInCatchVariables`, `alwaysStrict`. Nothing below is included.

**Catches wrong code**

- **`noUncheckedIndexedAccess`** — `arr[i]` and `obj[k]` become `T | undefined`. Highest-yield flag against agent-written code. Fixing the resulting errors can change runtime behavior (an added guard is a new branch), so run the tests.
- **`noPropertyAccessFromIndexSignature`** — index-signature properties must be read as `obj["k"]`. With the above, "declared property" and "dynamic lookup" become syntactically distinct, so a hallucinated property name is a compile error instead of `undefined` at runtime.
- **`exactOptionalPropertyTypes`** — `?:` stops accepting explicit `undefined`; `{ a: undefined }` and `{}` stop being interchangeable.
- **`noImplicitOverride`** — a renamed base method becomes an error, not a silently orphaned override.
- **`noImplicitReturns`** / **`noFallthroughCasesInSwitch`** — all branches return when any does; no silent fallthrough.
- **`noUnusedLocals` / `noUnusedParameters`** — catches half-applied refactors, a characteristic agent failure. Prefix genuinely-unused interface params with `_` rather than disabling the flag.
- **`allowUnreachableCode: false` / `allowUnusedLabels: false`** — errors, not editor-only hints.
- **`noUncheckedSideEffectImports`** — default in 7.0; keep it written. A typo'd `import "./setup"` is otherwise silently ignored.

**Legible without inference**

- **`isolatedDeclarations`** — an export whose declaration can't be produced from the local file alone must be annotated. Trivial cases still infer (`export const n = 1`), so this is not literally "annotate everything"; what it removes is cross-file inference at the module boundary. Costs return types on non-trivial exports; buys a public API an agent can read from one file, compiler-enforced. The flagship "type instead of comment" flag. It does not by itself speed up `tsc -b` — the parallelism it enables requires a separate syntactic declaration-emit step (oxc, rolldown, or a `noCheck` phase).
- **`moduleDetection: "force"`** — every file is a module; removes the case where a file with no imports quietly shares global scope.
- **`types: [...]`** — every ambient global has one declared source. Otherwise `describe` or `process` appears from nowhere and no grep explains it. Also 20–50% faster builds.
- **`verbatimModuleSyntax`** — imports emit as written; type-only imports must say `import type`.
- **`erasableSyntaxOnly`** — bans `enum`, runtime `namespace`, parameter properties, `import =`/`export =`, `<T>x`. Each hides runtime behavior behind type-looking syntax. Replacements for `enum` (`as const` object, literal union, `Map`) differ in runtime shape — pick deliberately and test. **Skip** in frameworks built on parameter properties (NestJS constructor injection, TypeORM), and see [§6](#6-commonjs).
- **`useDefineForClassFields`** — implicitly true at `target` ≥ ES2022, but it changes field-initialization semantics, so pin it.
- **`allowJs: false`** — keep JS out. If unavoidable, set `checkJs: true`; 7.0 reworked JSDoc handling, so JSDoc-typed JS is more brittle than before.

**Consumable output**

- **`noErrorTruncation`** — `tsc` truncates long types with `... N more ...` by default. An agent handed a truncated type can't fix the error and will guess. Zero cost, easy to forget because it doesn't affect checking. It does not appear in `--showConfig` even when set — verify it behaviorally with a long-union error.
- **`--pretty false`** on agent-facing runs — one error per line, no ANSI, greppable.

## Compile-Success Is Not Runtime-Success

`tsc` exiting 0 proves the types check. It does not prove the emitted JS loads. The gap opens wherever the compiler's resolver is more permissive than the runtime's: `moduleResolution: bundler` with Node-loaded output, extensionless relative imports, an `imports` alias mapped at source paths, an `exports` map whose `types` condition points somewhere the runtime doesn't.

Every route that emits therefore ends with an execution probe, not a compile:

- §2 — `node dist/index.js` after a clean build.
- §3 — `npm pack`, install the tarball into a scratch consumer, `import` it under both `node` and a `nodenext` `tsc`, and check `--traceResolution` if either fails.
- §4 — build with `tsc -b`, then run the package that declares the `references` edge, so its emitted import of the dependency resolves at runtime — in [§4](#4-monorepo-with-project-references)'s graph, run `app`, which imports `core`.

An agent that skips these will ship a package that passed every gate in this file and throws on first import.

## Path Aliases: Relative `.js` First

For anything Node loads, plain relative `./thing.js` imports are the default. Aliases add a second resolver that has to agree with the first, and nothing checks that it does.

When an alias is genuinely wanted, use `package.json` `imports` rather than tsconfig `paths` — `imports` is resolved by Node itself and honored by `tsc` under both `nodenext` and `bundler`, so there is one enforced source of truth, whereas `paths` is understood only by `tsc`. Two rules:

```json
{ "type": "module", "imports": { "#src/*": "./dist/*" } }
```

- **Map to the output, not the source.** A package that runs from `dist/` and maps `"#/*": "./src/*"` will resolve at runtime into `src/`, where the `.js` files don't exist. For `noEmit` bundled apps the source mapping is fine, because the bundler resolves before anything runs.
- **The bare `#/` prefix needs Node ≥ 24.14.0** (or ≥ 25.4.0 on the current line), where `module: allow subpath imports that start with #/` landed (nodejs/node#60864). On older Node the runtime throws `ERR_INVALID_MODULE_SPECIFIER` while `tsc` 7.0.2 exits 0 — verified both ways. The TypeScript 6.0 announcement attributes the feature to "newer Node.js 20 releases"; the Node changelog says 24.14.0. Use a segment prefix like `#src/*` unless you can pin ≥ 24.14.0.

`baseUrl` is removed in 7.0; if `paths` is unavoidable, entries are now relative to the project root.

## Scripts

Single-project (§1–§3). Monorepos use the build-mode scripts in [§4](#4-monorepo-with-project-references) instead.

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit --pretty false --checkers 4",
    "typecheck:watch": "tsc --noEmit --watch --pretty false --checkers 4",
    "typecheck:test": "tsc -p tsconfig.test.json --pretty false --checkers 4",
    "typecheck:ci": "tsc -p tsconfig.ci.json --pretty false --checkers 4",
    "check:decl": "tsc -p tsconfig.declarations.json --pretty false --checkers 4"
  }
}
```

The same `--pretty false` and `--checkers N` on every one of them: an agent reads all of these, so any script that keeps ANSI output or floats its checker count is an inconsistency the agent has to absorb. When `typecheck` already targets another tool, leave it and add `typecheck:tsc` alongside.

`--noEmit` is only meaningful in `typecheck` for a route whose base doesn't already set it; for §2 and §3 it overrides an emitting config, which is what you want in the loop.

## Determinism and Caching Hazards

Non-determinism hurts an agent more than a human: an agent treats every run as ground truth.

- **Pin `--checkers`** (default 4; the flag, like `--builders`, is still marked experimental in 7.0). Varying the worker count can rarely surface order-dependent results. `--singleThreaded` isolates a suspected ordering issue.
- **`stableTypeOrdering`** is always on in 7.0 and stabilizes internal type and symbol ordering, so 6.0-vs-7.0 output diffs are expected rather than a regression. It is not a guarantee that declaration output is independent of declaration order.
- **Bypass the cache at the gate** — `incremental: false` for non-composite, `tsc -b --force` for composite. `tsc -b` does track `.d.ts` changes normally; this is defense against restored VCS timestamps, hand-deleted outputs, and compiler bugs.
- **Clear stale build state with `tsc -b --clean`**, which is cross-platform and scoped to the project graph. Reach for a manual recursive delete only if that fails, and then enumerate what will be removed before removing it.

## The Gate

`tsc --noEmit` is the loop. The gate catches what `tsc` structurally cannot see:

- `npm run typecheck:ci` — full check, no cache, `skipLibCheck: false`
- `npm run typecheck:test` — the files the main config excludes
- `npm run check:decl` — `isolatedDeclarations`
- A **runtime probe** per [Compile-Success Is Not Runtime-Success](#compile-success-is-not-runtime-success) — the only gate that catches the resolver gap
- `oxlint --type-aware --type-check` — type-aware rules, notably `no-floating-promises` and the `no-unsafe-*` family; `tsc` has no opinion on an `any` crossing a boundary. Needs `oxlint-tsgolint` (versioned in lockstep with TS 7.0.x) and TS 7.0+, shares one program with the lint pass, and can subsume the separate `tsc --noEmit` step in CI. Configure in the companion oxlint skill.
- `attw --pack .` + `publint --strict` for libraries — mechanically verify the `exports` map instead of eyeballing it
- `type-coverage --at-least 99 --strict` — a number in CI instead of a belief about leftover `any`. Its peer range admits `typescript@7`, but 7.0 ships no JS API and it crashes at load — point it at the 6.0 alias.

## Comments → Checked Artifacts

When you find the left column, replace it with the right column.

| Rotting comment | Checked replacement |
| --- | --- |
| `// @ts-ignore` | `// @ts-expect-error <reason>` — errors once fixed, so it self-deletes |
| `// this is a user id, not a name` | branded type: `string & { readonly brand: unique symbol }` |
| `// one of: 'a' \| 'b' \| 'c'` | literal union, or `as const` object + `(typeof X)[keyof typeof X]` |
| `// handle new variants here too` | `default: return assertNever(x)` |
| `// keep in sync with the API schema` | derive it: `z.infer<typeof S>`, `keyof`, `Extract`, mapped types |
| `// keep in sync with the config` | `satisfies Config` on the literal |
| `// this cast is safe because …` | a validator that actually checks, plus tests — a hand-written `x is T` predicate can still lie, so prefer an inferred predicate or a schema parse over asserting one |
| `// returns null if not found` | put it in the return type: `T \| null` |
| `// this function's shape is …` | `isolatedDeclarations` makes the signature mandatory |
| `// subtle overload, don't break it` | type-level test: `expectTypeOf` (vitest) or `tsd` in `*.test-d.ts` |

Keep two comment forms, because they are machine-consumed: `@deprecated` and `@ts-expect-error`.

## When a Setting "Isn't Working"

```bash
npx tsc --showConfig      # merged config after all `extends` — run before guessing
npx tsc --explainFiles    # why each file is in the program
npx tsc --listFilesOnly   # what the program actually contains
```

`--showConfig` prints merged explicit values only — 7.0 expands neither defaults nor output-formatting options (`noErrorTruncation` is silently dropped even when set), so verify those behaviorally. On a solution root, `--listFilesOnly` returning nothing is the expected symptom of the §4 traversal problem, not an empty project.

## Merge Rules for an Existing tsconfig

Do not overwrite. Surface the diff against the matching template, flag standing-policy violations, let the user decide. Keep each step a separate reviewable change with `npm run typecheck` green before the next — propose the commits, don't make them unasked:

0. **TS 7 hard errors** — nothing else is evaluable until the config loads. `npx @andrewbranch/ts5to6 --fixBaseUrl <config>` and `--fixRootDir <config>` handle the mechanical `baseUrl`/`rootDir` parts, one mode per run, following `references` and `extends` from the config you point at. Diff the result before keeping it.
1. `strict: true`
2. Explicit `types`, `rootDir`; pinned `target` / `lib` / `module` / `moduleResolution` — and confirm the trio row matches who actually resolves the output
3. `noErrorTruncation` + `--pretty false` + `--checkers N` — before the noisy steps, so their errors arrive readable
4. `noUnusedLocals` + `noUnusedParameters`
5. `noImplicitReturns` + `noFallthroughCasesInSwitch`
6. `noUncheckedIndexedAccess` — expect the largest error count
7. `noPropertyAccessFromIndexSignature`
8. `exactOptionalPropertyTypes`
9. `verbatimModuleSyntax` + `isolatedModules` + `moduleDetection: force`
10. `erasableSyntaxOnly` — mostly `enum` migration; check runtime shape
11. Split out the §5 gate configs, including `tsconfig.test.json`
12. `isolatedDeclarations` — largest diff, mostly additive annotations, so do it last

Steps 6 and 12 produce hundreds of errors on a codebase that has never had them. Both grind through incrementally; step 6 in particular can change behavior, so keep the tests running.

## Verification

- `npx tsc --showConfig -p <each config>` matches the two-tier table — check resolved values per file, not the snippets.
- `npm run typecheck`, `typecheck:test`, `typecheck:ci` (with no `.tsbuildinfo` present), and `check:decl` all exit 0.
- `typecheck:ci` leaves no build artifacts behind; `check:decl` leaves no `.tsbuildinfo`.
- The emitted program actually runs — §2 executes, §3 installs from `npm pack` into a scratch consumer and resolves both JS and types, §4 imports across packages.
- On a monorepo, introduce a deliberate type error in one leaf and confirm the root gate exits non-zero. If it exits 0, the gate isn't traversing references.
- A deliberate `const x: string = arr[0]` errors — confirms `noUncheckedIndexedAccess` is live.
- An error containing a long type shows it in full, not `... N more ...` — confirms `noErrorTruncation`.
- With `declarationMap: true`, `.d.ts.map` files appear next to the `.d.ts` (verified on 7.0.2); early-7.0 reports of gappy maps mean spot-check a jump-to-definition rather than assuming their content.

## References

- TypeScript compiler options: <https://www.typescriptlang.org/tsconfig>
- Announcing TypeScript 7.0: <https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/>
- TypeScript 6.0 deprecations and new defaults: <https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/>
- Choosing compiler options (library authors): <https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options>
- Project references: <https://www.typescriptlang.org/docs/handbook/project-references>
- Node.js subpath imports: <https://nodejs.org/api/packages.html#subpath-imports>
- Node.js 24.14.0 release notes (bare `#/` subpath imports): <https://nodejs.org/en/blog/release/v24.14.0>
- Oxlint type-aware linting: <https://oxc.rs/docs/guide/usage/linter/type-aware.html>
