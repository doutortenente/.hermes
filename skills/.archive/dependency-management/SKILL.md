---
name: dependency-management
description: "Use when updating Node/TypeScript package dependencies."
---

# Dependency Management

## Core Principle

Treat dependency updates like medication changes: verify compatibility before applying, measure effect after, and always have a rollback. A `pnpm update --latest` that breaks the build is a failed intervention.

## Safe Upgrade Workflow

1. **Snapshot baseline** — `pnpm check` (typecheck + lint + test) must pass before any change.
2. **Read changelogs** — For major version bumps (especially TypeScript, React, Next.js, ESLint), scan release notes for breaking changes.
3. **Update selectively** — `pnpm up <pkg>@<version>` for targeted upgrades; avoid blanket `--latest` on projects with strict tooling chains.
4. **Re-verify** — Run `pnpm check` after each logical group of changes.
5. **Commit lockfile** — `pnpm-lock.yaml` changes go in the same commit as `package.json`.

## Known Compatibility Matrix (update as encountered)

| Toolchain | Max Compatible TypeScript | Notes |
|---|---|---|
| `typescript-eslint@8.x` | `6.x` | TS 7.0 breaks `typescript-eslint` — see [references/typescript-eslint-ts7.md](references/typescript-eslint-ts7.md) |
| `eslint@10.x` + `eslint-plugin-react@7.37.x` | — | Incompatible pair; `eslint-plugin-react@7.37.x` needs ESLint 9 |

## Pitfalls

- **Blind `--latest`** — Pulls major versions that cascade into peer-dep conflicts.
- **Ignoring peer warnings** — `pnpm peers check` surfaces real breaks; treat as errors.
- **Skipping `pnpm check`** — Typecheck is the canary; lint/test may have pre-existing noise.

## Verification Standard

After any dependency change, `pnpm typecheck` must pass. `pnpm lint` and `pnpm test` should not regress (new failures = blocked).

## Reference Guides

- **TypeScript 7 + typescript-eslint 8 incompatibility** → [references/typescript-eslint-ts7.md](references/typescript-eslint-ts7.md)
  **MUST read when** `pnpm update` pulls TypeScript 7.x and `eslint` crashes with "typescript-eslint does not support TS 7.0".
