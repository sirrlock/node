# @sirr/node — Claude Development Guide

## Purpose

Node.js client and npx CLI for the Sirr HTTP API.
Published to npm as `@sirr/node`. Zero production dependencies — uses native `fetch`.

## What Lives Here

- `src/index.ts` — `SirrClient` class (the library)
- `src/cli.ts` — CLI entrypoint (`npx @sirr/node push ...`)
- `src/index.test.ts` — 14 unit tests (all methods, fetch mocked)

## API Surface

```typescript
class SirrClient {
  constructor(opts: { server?: string; token: string })

  push(key: string, value: string, opts?: { ttl?: number; reads?: number }): Promise<void>
  get(key: string): Promise<string | null>   // null if burned/expired
  delete(key: string): Promise<boolean>
  list(): Promise<SecretMeta[]>
  pullAll(): Promise<Record<string, string>>
  withSecrets<T>(fn: () => Promise<T>): Promise<T>
  prune(): Promise<number>
  health(): Promise<{ status: string }>
}
```

## Stack

- TypeScript, Node 18+
- Native `fetch` — no axios, no node-fetch
- `tsc` for build (CommonJS output)
- `jest` + `ts-jest` for tests

## Key Rules

- `get()` returns `null` on 404 — never throw for not-found
- All other non-2xx responses throw `SirrError`
- Never log secret values
- `withSecrets()` must restore original env on exit, even on exception
- Keep zero production dependencies

## Commands

```bash
npm install       # install deps
npm run build     # tsc → dist/
npm test          # jest (14 tests)
```

## Relationship to sirr/

This repo was extracted from `sirr/packages/node/`. The MCP server (`@sirr/mcp`)
remains in the [SirrVault/sirr](https://github.com/SirrVault/sirr) monorepo
because it is co-released with the server binary. This client has an independent
release cadence once the HTTP API stabilises.

## Pre-Commit Checklist

Before every commit and push, review and update if needed:

1. **README.md** — Does it reflect new methods or behavior?
2. **CLAUDE.md** — New constraints or API decisions worth recording?
