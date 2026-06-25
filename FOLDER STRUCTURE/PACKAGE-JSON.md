# package.json — Workspace Script Standard

> pnpm workspace only. Root scripts orchestrate apps and packages; each app/package owns its local scripts.

## TL;DR

```text
root package.json      → workspace commands
apps/*/package.json    → app-specific scripts
packages/*/package.json → package build/test scripts
pnpm-workspace.yaml    → workspace membership
```

---

## Root `package.json`

```json
{
  "private": true,
  "packageManager": "pnpm@9.1.0",
  "scripts": {
    "dev": "pnpm -r --parallel dev",
    "build": "pnpm -r build",
    "lint": "pnpm -r lint",
    "test": "pnpm -r test",
    "typecheck": "pnpm -r typecheck",
    "format": "prettier --write ."
  }
}
```

---

## Workspace File

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

---

## App Scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "lint": "next lint",
    "typecheck": "tsc --noEmit"
  }
}
```

Backend apps can replace `next` scripts with `tsx`, `tsup`, or `tsc` based commands.

---

## Common Commands

```bash
pnpm dev
pnpm build
pnpm --filter web-host dev
pnpm --filter api lint
pnpm --filter @kt-events/ui build
```

---

## Rules

1. Keep orchestration scripts in the root `package.json`.
2. Keep app/package implementation scripts inside their own `package.json`.
3. Use `pnpm --filter <name>` for one app or package.
4. Do not use Turbo; pnpm workspace scripts are the standard.
