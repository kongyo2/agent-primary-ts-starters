---
name: ts-npm-zod-starter
description: Add zod to a TypeScript npm project and install the boundary-only parsing convention. Use when starting a new TS project that needs runtime schema validation, or when layering zod onto an existing TS npm project — even if the user doesn't explicitly say "zod" or "validation".
---

# TS + npm Zod Starter

Use this when adding zod to a TypeScript npm project. In a monorepo, edit the workspace the user names — ask if it isn't named.

## Expected Outcome

- `zod` is in `dependencies` of the target `package.json`.
- The Boundary Parsing Policy below is the documented convention for the project's zod usage.

## Installation

```bash
npm add zod
```

zod ships runtime code, so it belongs in `dependencies` (not `devDependencies`). When zod is already declared, keep its current version unless the user asks for a change.

## Boundary Parsing Policy

- Parse untrusted input at the boundary with `z.parse` / `z.safeParse`. Downstream code takes the parsed value as trusted and does not re-parse.
- `z.infer<typeof Schema>` is the single source of truth for the static type — let it stand alone instead of declaring a parallel `interface` or `type` alias.

Why: parsing at every layer doubles the work and obscures where the trust boundary actually is.

## Schemas Directory

When the project has a concrete shape worth validating, add its schema under `src/schemas/<name>.ts`, exporting the schema, its `z.infer` type, and a `parse(raw: unknown)` wrapper used at the boundary.

When `src/schemas/` already exists, defer to the project's existing convention.

## Verification

- `npx tsc --noEmit` passes.
- `zod` appears under `dependencies` in `package.json`.

## References

- zod docs: <https://zod.dev/>
