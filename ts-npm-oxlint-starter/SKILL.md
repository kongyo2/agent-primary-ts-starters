---
name: ts-npm-oxlint-starter
description: Add Oxlint to a TypeScript npm project with agent-friendly defaults that focus error-level diagnostics on real bugs, silence stylistic noise, and emit output in oxlint's `agent` format for downstream LLM consumption. Use when bootstrapping a new TS project that needs linting, or when adding Oxlint to an existing TS npm project — even if the user doesn't explicitly say "oxlint" or "linter".
---

# TS + npm Oxlint Starter (Agent-Friendly)

Use this when adding Oxlint to a TypeScript npm project. In a monorepo, edit the workspace the user names — ask if it isn't named.

The defaults below treat **the LLM agent as the primary consumer of lint output**, not human eyes. Stylistic and pedantic warnings during agent edit loops are pure noise that crowds out real bugs; formatting belongs to a formatter, not a linter. Oxlint ships a dedicated `agent` value for `--format`, which the scripts below wire into the default `lint` command.

## Expected Outcome

- `oxlint` is in `devDependencies` of the target `package.json`.
- `.oxlintrc.json` exists with the agent-friendly category and rule settings below.
- `package.json` scripts include `lint`, `lint:fix`, and `lint:strict`.

## Installation

```bash
npm add -D oxlint
```

When oxlint is already declared, keep its current version unless the user asks for a change.

## Agent-Friendly Configuration

Create `.oxlintrc.json` if it does not exist:

```json
{
  "plugins": ["typescript", "import", "promise", "node"],
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

Why each value is chosen for an agent-primary workflow:

- **`categories.style: off` / `pedantic: off`** — stylistic and pedantic warnings during agent edit loops drown out real bugs and waste the inner feedback loop. Style is the formatter's domain (e.g. Prettier), not the linter's.
- **`categories.correctness: error`** — code that is definitely wrong should block, so the agent fixes it before moving on.
- **`categories.suspicious: warn` / `perf: warn`** — informative without halting; the agent decides.
- **`rules.no-console: off`** — agent debugging routinely inserts `console.log` calls; flagging them slows the inner loop. Strip them at release time with a separate gate, not on every lint.
- **`plugins`** — listed explicitly because setting `plugins` in oxlint **overwrites the default set** rather than extending it. `typescript`, `import`, `promise`, `node` cover the typical TS/Node project surface.

When `.oxlintrc.json` already exists, keep it. Surface the diff against the values above so the user can decide whether to migrate, but do not overwrite.

## Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "lint": "oxlint --format agent",
    "lint:fix": "oxlint --fix",
    "lint:strict": "oxlint --deny-warnings"
  }
}
```

Why each script:

- **`lint`** uses `--format agent`, an oxlint output format the oxc team designed explicitly for LLM agent consumption.
- **`lint:fix`** applies auto-fixable rule corrections in one pass.
- **`lint:strict`** promotes warnings to a non-zero exit code for CI gating.

When `lint` already targets a different linter, leave it untouched and add `lint:oxlint` alongside.

## Verification

- `npm run lint` exits 0 on a clean repository.
- `.oxlintrc.json` parses as valid JSON.

## References

- Oxlint config: <https://oxc.rs/docs/guide/usage/linter/config>
- Oxlint CLI: <https://oxc.rs/docs/guide/usage/linter/cli>
