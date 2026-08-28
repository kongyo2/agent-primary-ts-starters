# agent-primary-ts-starters

> Agent-primary TypeScript + npm skills. Defaults tuned for LLM-agent machine-verifiability and machine-explorability — diff stability, line-number stability, inner-loop speed, and parseable tool output over human-readable conventions.

Two `SKILL.md` guides built on one premise: the reader, editor, and primary consumer of a modern TS codebase is a coding agent. Mistakes should snag on the type checker, the tests, and the linter — not on a human reading linearly — and information worth keeping lives in checked elements (types, `satisfies`, tests), not in comments that no one maintains.

## Skills

| Skill | What it does |
| --- | --- |
| [`agent-primary-ts-starter`](./agent-primary-ts-starter/SKILL.md) | The consolidated starter: TypeScript 7 `tsconfig.json` (maximally strict, machine-verifiable, per-use-case routes with two-tier loop/gate configs and runtime probes), Prettier with diff-minimizing settings, Oxlint focused on real bugs with `--format agent` output, and the npm scripts wiring them together. Deep tsconfig doctrine lives in [`references/tsconfig.md`](./agent-primary-ts-starter/references/tsconfig.md). |
| [`ts-comment-purge`](./ts-comment-purge/SKILL.md) | Unconditional comment removal via [`@kongyo2/ts-comment-scanner`](https://www.npmjs.com/package/@kongyo2/ts-comment-scanner): AST-based, strings/templates/regex/JSX never touched, directives and license headers preserved, `--diff` scoping for post-agent-session cleanup. |

## Quickstart

The companion CLI [`@kongyo2/apts`](https://www.npmjs.com/package/@kongyo2/apts) lets agents semantically search and retrieve these skills on demand:

```shell
# Find the right skill for the task
npx @kongyo2/apts@latest search "set up strict tsconfig"

# Pull a guide into context
npx @kongyo2/apts@latest retrieve agent-primary-ts-starter

# List the whole catalog
npx @kongyo2/apts@latest list
```

See the [`@kongyo2/apts` README](https://github.com/kongyo2/agent-primary-ts-starters-src#readme) for the full CLI reference.

## Install as a Claude Code plugin

This repo is also a Claude Code plugin marketplace. From inside Claude Code:

```
/plugin marketplace add kongyo2/agent-primary-ts-starters
/plugin install agent-primary-ts-starters@agent-primary-ts-starters
```

## License

MIT — see the [source repo](https://github.com/kongyo2/agent-primary-ts-starters-src/blob/main/LICENSE) for the full text.
