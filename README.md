# agent-primary-ts-starters

> Agent-primary TypeScript + npm starter skills (tsconfig / Prettier / Oxlint / Zod). Defaults tuned for LLM-agent diff stability, line-number stability, and inner-loop speed over human-readable conventions.

Four `SKILL.md` guides that bootstrap a TypeScript npm project the way coding agents want it — stable line numbers across edits, single-line diffs, fast `tsc --noEmit` feedback, and parseable lint output.

## Skills

| Skill | What it sets up |
| --- | --- |
| [`ts-tsconfig-modern-strict-starter`](./ts-tsconfig-modern-strict-starter/SKILL.md) | TypeScript 7 `tsconfig.json`, maximally strict and machine-verifiable: per-use-case `target`/`module`/`moduleResolution` rows, two-tier loop/gate configs, and runtime probes so a green `tsc` can't hide runtime-dead output. |
| [`ts-npm-prettier-starter`](./ts-npm-prettier-starter/SKILL.md) | Prettier with diff-minimizing defaults (wider `printWidth`, all-trailing-commas, etc.) so single-line agent edits don't reflow surrounding code. |
| [`ts-npm-oxlint-starter`](./ts-npm-oxlint-starter/SKILL.md) | Oxlint focused on real bugs, with stylistic noise silenced and output in `--format agent` for downstream LLM consumption. |
| [`ts-npm-zod-starter`](./ts-npm-zod-starter/SKILL.md) | Zod with a documented boundary-only parsing convention. |

## Quickstart

The companion CLI [`@kongyo2/apts`](https://www.npmjs.com/package/@kongyo2/apts) lets agents semantically search and retrieve these skills on demand:

```shell
# Find the right skill for the task
npx @kongyo2/apts@latest search "set up strict tsconfig"

# Pull a guide into context
npx @kongyo2/apts@latest retrieve ts-tsconfig-modern-strict-starter

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
