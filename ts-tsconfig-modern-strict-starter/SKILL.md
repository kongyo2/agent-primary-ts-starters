---
name: ts-tsconfig-modern-strict-starter
description: Set up a TypeScript tsconfig.json with modern (ESNext + bundler/nodenext) and maximally strict settings, tuned so that `tsc --noEmit` becomes a fast, reliable inner feedback loop for the LLM agent doing the editing. Use whenever a tsconfig.json is being created, edited, reviewed, troubleshot, or even just discussed — apply the standing policy even when the user only asks about a single option. Default to this skill instead of free-recall whenever the answer involves a tsconfig field.
---

# TS + npm tsconfig Modern Strict Starter (Agent-Friendly)

Use whenever a `tsconfig.json` is being created, edited, reviewed, or troubleshot in a TypeScript npm project. In a monorepo, edit the workspace the user names — ask if it isn't named.

The standing policy below is tuned so that **`tsc --noEmit` is the LLM agent's primary inner feedback loop**. Modern syntax + maximum strictness narrows the surface where an agent can ship a silent bug; explicit module resolution removes whole classes of "why isn't this importing" turns that would otherwise burn agent context; `incremental: true` keeps the loop fast enough to run between each edit.

## Standing Policy (override only with a stated reason)

- `"target": "esnext"`. Downleveling is the bundler / runtime's job, not tsc's.
- `"strict": true` plus every recommended add-on flag below.
- `"moduleResolution": "bundler"` for app code via a bundler; `"nodenext"` for code running on Node directly.
- `"skipLibCheck": true`. Always — a single broken `.d.ts` in `node_modules` otherwise stops the entire feedback loop.
- `"incremental": true` (or `"composite": true` under project references). The `.tsbuildinfo` cache makes the second-and-onward `tsc --noEmit` run near-instant, which is what keeps the agent loop tight enough to run between edits.
- `"isolatedModules": true` and `"verbatimModuleSyntax": true` for any code touched by esbuild / SWC / Babel / Vite.

When the user proposes loosening any of these, ask what concrete problem they are solving before agreeing.

## Pick the Use Case First

Resolve the use case **before** writing or modifying a tsconfig — the four templates differ in incompatible ways.

```
Is tsc emitting JS?
├── No, a bundler emits  ─────────────────► §1  Browser app + bundler
└── Yes, tsc emits
    ├── Running on Node directly (ESM)  ──► §2  Node ESM app
    └── Publishing to npm
        ├── Single package           ────► §3  Library
        └── Monorepo                 ────► §4  Project references
```

## The `target` / `module` / `moduleResolution` Trio

Co-dependent. Pick one row, don't improvise.

| Scenario                              | `target`  | `module`     | `moduleResolution` |
| ------------------------------------- | --------- | ------------ | ------------------ |
| Browser app via bundler               | `esnext`  | `esnext`     | `bundler`          |
| Node.js native ESM                    | `esnext`  | `nodenext`   | `nodenext`         |
| Publishable library (build via tsc)   | `es2020`+ | `esnext`     | `bundler`          |
| Node.js CommonJS (legacy)             | `es2022`  | `commonjs`   | `node`             |

Common mismatches:

- `module: "commonjs"` + `moduleResolution: "bundler"` — invalid combination.
- `nodenext` + extensionless relative imports — Node ESM requires `.js` on relative imports even when the source is `.ts`.

## §1. Browser app + bundler (Vite, Next.js, Rspack, etc.)

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["esnext", "dom", "dom.iterable"],
    "jsx": "react-jsx",

    "noEmit": true,
    "incremental": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,

    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "allowUnreachableCode": false,
    "allowUnusedLabels": false
  },
  "include": ["src"]
}
```

Drop `"dom"` and `"dom.iterable"` from `lib` for non-browser code. Change `jsx` to `"preserve"` plus `jsxImportSource` for Preact / Solid.

## §2. Node.js ESM application

`package.json` must have `"type": "module"`. Relative imports require `.js` even when the source is `.ts`.

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["esnext"],

    "outDir": "./dist",
    "rootDir": "./src",
    "sourceMap": true,
    "incremental": true,

    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts", "dist"]
}
```

## §3. Publishable library

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",

    "rootDir": "./src",
    "outDir": "./dist",
    "declarationDir": "./dist-types",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "incremental": true,
    "allowImportingTsExtensions": false,

    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts"]
}
```

When publishing, also configure `package.json` `exports` — the `types` field must come first inside each conditional block.

## §4. Monorepo with project references

Root `tsconfig.json`:

```json
{
  "files": [],
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/utils" },
    { "path": "./packages/app" }
  ]
}
```

Each package extends a shared base and sets `composite: true`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true
  },
  "include": ["src/**/*"],
  "references": [{ "path": "../core" }]
}
```

Build with `tsc --build` (or `tsc -b`). Plain `tsc` against the root won't resolve references.

## Strict Flags Beyond `strict: true`

These are not enabled by `strict: true` but should be on by default for an agent-primary workflow:

- **`noUncheckedIndexedAccess`** — `arr[i]` and `obj[k]` become `T | undefined`. Catches a class of silent index bugs that agents emit easily.
- **`exactOptionalPropertyTypes`** — `?:` no longer accepts an explicit `undefined`.
- **`noImplicitOverride`** — subclass overrides must say `override`.
- **`noImplicitReturns`** — all branches return when any branch does.
- **`noFallthroughCasesInSwitch`** — no silent fallthrough.
- **`noUnusedLocals` / `noUnusedParameters`** — agent leftovers get caught at type-check. For unused parameters in interface implementations, prefix with `_` instead of disabling the flag globally.
- **`allowUnreachableCode: false`** / **`allowUnusedLabels: false`** — treat as errors, not warnings.

When the toolchain supports it, also add `verbatimModuleSyntax` (any non-tsc transpiler), `isolatedDeclarations` (TS 5.5+ library), `erasableSyntaxOnly` (TS 5.8+ type-strip runtime).

## Agent Inner Loop

Add to `package.json` so the agent has a single command for the type-check feedback loop:

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "typecheck:watch": "tsc --noEmit --watch"
  }
}
```

With `incremental: true` set, the second-and-later `npm run typecheck` returns in well under a second on typical projects — which is what makes this usable as an inner loop the agent runs between edits. When `typecheck` already targets another tool, leave it untouched and add `typecheck:tsc` alongside.

## When a Setting "Isn't Working", Show the Resolved Config

```bash
npx tsc --showConfig
```

Prints the fully resolved config after all `extends`. The single most useful command when a setting "isn't working" — run it before guessing.

## Merge Rules for an Existing tsconfig

When `tsconfig.json` already exists, do not overwrite. Surface the diff against the matching template above, flag any standing-policy violations (`strict: false`, `skipLibCheck: false`, trio mismatch), and let the user decide whether to migrate. Recommended migration order to minimize churn:

1. `strict: true`
2. `noUnusedLocals` + `noUnusedParameters`
3. `noImplicitReturns` + `noFallthroughCasesInSwitch`
4. `noUncheckedIndexedAccess`
5. `exactOptionalPropertyTypes`
6. `verbatimModuleSyntax`

Run `npm run typecheck` after each step. Commit between steps.

## Verification

- `npx tsc --noEmit` exits 0 on the project after the chosen template is applied (or surfaces real type errors that should be fixed).
- `npx tsc --showConfig` prints a resolved config matching the standing policy.

## References

- TypeScript compiler options: <https://www.typescriptlang.org/tsconfig>
- TypeScript modules guide: <https://www.typescriptlang.org/docs/handbook/modules/introduction.html>
