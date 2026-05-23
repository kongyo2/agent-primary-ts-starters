---
name: ts-npm-prettier-starter
description: Add Prettier to a TypeScript npm project with agent-friendly defaults that minimize diff size and keep line numbers stable across LLM-driven edits. Use when bootstrapping a new TS project that needs auto-formatting, or when adding Prettier to an existing TS npm project — even if the user doesn't explicitly say "prettier" or "formatter".
---

# TS + npm Prettier Starter (Agent-Friendly)

Use this when adding Prettier to a TypeScript npm project. In a monorepo, edit the workspace the user names — ask if it isn't named.

The defaults below treat **the LLM agent as the primary editor**, not human eyes. Prettier's human-tuned defaults (printWidth 80, etc.) reformat large regions when an agent edits a single line, shift surrounding line numbers, and silently invalidate `old_string` matches the agent is holding for upcoming edits in the same session. These settings shrink that failure surface.

## Expected Outcome

- `prettier` is in `devDependencies` of the target `package.json`.
- `.prettierrc.json` exists with the agent-friendly settings below.
- `.prettierignore` exists covering `node_modules` and common build artifacts.
- `package.json` scripts include `format` (write) and `format:check` (read-only check).

## Installation

```bash
npm add -D prettier
```

When prettier is already declared, keep its current version unless the user asks for a change.

## Agent-Friendly Configuration

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

Why each value is chosen for an agent-primary workflow:

- **`printWidth: 120`** — wider than the default 80, so a one-line agent edit rarely triggers auto-wrap. Stable surrounding line counts keep both `old_string` matches and conversational line-number references valid.
- **`trailingComma: "all"`** — appending an item to an array, object, or argument list becomes a single added-line diff instead of a two-line diff that also rewrites the previous line's `,`.
- **`arrowParens: "always"`** — adding a type annotation to an arrow parameter doesn't also force inserting parens, so the annotation edit stays a localized change.
- **`semi: true`** — ASI quirks don't silently change behavior when an agent prepends a line that starts with `[`, `(`, `+`, `-`, or `/`.
- **`endOfLine: "lf"`** — avoids whole-file CRLF reformat noise when Windows and Unix contributors share the repo.

When `.prettierrc.*` already exists, keep it. Surface the diff against the values above so the user can decide whether to migrate, but do not overwrite.

## Ignore Patterns

Create `.prettierignore` if it does not exist:

```
node_modules
dist
build
coverage
*.min.js
package-lock.json
*.md
```

`*.md` is in the list because Prettier's markdown formatter reflows paragraphs, list spacing, and table column widths across edits, which breaks `old_string` matches the same way reformatted JS/TS would — and markdown is content, not code, so the formatting consistency isn't worth the agent-edit cost.

When `.prettierignore` already exists, append only the missing entries from the list above and leave existing entries in place.

## Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

When `format` already targets a different formatter, leave `format` untouched and add `format:prettier` alongside it.

## Verification

- `npm run format` runs without error across the repository.
- `npm run format:check` exits 0 after a one-time `npm run format`.
- `.prettierrc.json` parses as valid JSON.

## References

- Prettier options: <https://prettier.io/docs/en/options>
