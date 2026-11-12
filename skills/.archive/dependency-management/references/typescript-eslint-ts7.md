# TypeScript 7.x + typescript-eslint 8.x Incompatibility

## Symptom

```
$ pnpm lint
Oops! Something went wrong! :(
ESLint: 10.9.1
Error: typescript-eslint does not support TS 7.0.
```

## Cause

`typescript-eslint@8.x` does not support TypeScript 7.0. The error originates from `typescript-eslint` trying to load the TS 7 API and failing.

## Fix

Pin TypeScript to latest 6.x:

```bash
pnpm add -D typescript@6.0.3  # or latest 6.x patch
pnpm install
```

## Verification

```bash
pnpm typecheck  # must pass
```

## Context

- Encountered: 2026-09-05 during `pnpm update --latest` in SASI-V3
- `typescript` jumped `6.0.3` → `7.0.2`
- `typescript-eslint` stayed `8.69.0` (latest at time)
- `pnpm typecheck` passes with TS 6.0.3; `pnpm lint` has separate pre-existing issue (ESLint 10 + eslint-plugin-react 7.37.5 incompatibility)

## Upstream Tracking

- typescript-eslint issue: https://github.com/typescript-eslint/typescript-eslint/issues/10940
- TS 7 announcement: https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/

## Prevention

- Add to `package.json` engines or `pnpm.overrides` if TS 7 must be avoided:
  ```json
  "pnpm": {
    "overrides": {
      "typescript": "6.0.3"
    }
  }
  ```
- Monitor typescript-eslint releases for TS 7 support before upgrading.