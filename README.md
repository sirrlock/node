# @sirr/sdk

**TypeScript client for ephemeral secrets. Built for AI-era workflows.**

`@sirr/sdk` is the Node.js / TypeScript client for [Sirr](https://github.com/SirrVault/sirr) — a self-hosted vault where every secret expires. Use it to give AI agents exactly the access they need, for exactly as long as they need it, with automatic cleanup.

---

## The Problem It Solves

Node.js AI frameworks — LangChain.js, Vercel AI SDK, OpenAI Agents, custom tool-calling loops — need credentials to do useful work: database connections, API keys, third-party tokens. Passing these as plain environment variables or hardcoded strings means they persist indefinitely in process memory, logs, and whatever storage the AI framework uses for context.

Sirr gives you a better primitive: **read-once, time-limited credentials** that delete themselves after use.

```typescript
// Give an agent a one-time DB connection — it reads, it's gone
await sirr.push('PROD_DB', connectionString, { reads: 1, ttl: 3600 })

// Agent fetches it — read counter hits 1 — record deleted on the server
const db = await sirr.get('PROD_DB')  // returns value
await sirr.get('PROD_DB')             // returns null — already burned
```

---

## Install

```bash
npm install @sirr/sdk
```

Requires Node 18+. Zero production dependencies — uses native `fetch`.

---

## Usage

```typescript
import { SirrClient } from '@sirr/sdk'

const sirr = new SirrClient({
  server: process.env.SIRR_SERVER ?? 'http://localhost:8080',
  token: process.env.SIRR_TOKEN!,
})

// Push a one-time secret
await sirr.push('API_KEY', 'sk-...', { ttl: 3600, reads: 1 })

// Retrieve — null if burned or expired
const value = await sirr.get('API_KEY')

// Pull all secrets into a plain object
const secrets = await sirr.pullAll()
// → { API_KEY: 'sk-...', DB_URL: 'postgres://...' }

// Inject all secrets as env vars for the duration of a callback
await sirr.withSecrets(async () => {
  // process.env.API_KEY is set here
  await runAgentTask()
  // env vars are cleaned up on return, even on throw
})

// Delete immediately
await sirr.delete('API_KEY')

// List (metadata only — values never returned by list)
const list = await sirr.list()

// Prune expired secrets
const { pruned } = await sirr.prune()
```

---

## AI Workflows

### LangChain.js tool with scoped credential

```typescript
import { DynamicTool } from 'langchain/tools'

const dbTool = new DynamicTool({
  name: 'query_database',
  description: 'Run a SQL query against the production database',
  func: async (query) => {
    // Credential pulled fresh each invocation — one-read budget enforced by Sirr
    const connStr = await sirr.get('AGENT_DB')
    if (!connStr) throw new Error('DB credential expired or burned')
    return runQuery(connStr, query)
  },
})
```

### Vercel AI SDK with burn-after-use key

```typescript
import { generateText } from 'ai'

// Push a scoped key before the agent run
await sirr.push('OPENAI_KEY', process.env.OPENAI_API_KEY!, { reads: 1, ttl: 300 })

// Agent task runs — key is consumed
const { text } = await generateText({
  model: openai('gpt-4o'),
  prompt: 'Summarize the meeting notes...',
})

// Whether or not we reach here, OPENAI_KEY burned after first read
```

### Inject secrets into a subprocess

```typescript
await sirr.withSecrets(async () => {
  // All Sirr secrets are set as process.env for this block
  await execa('python', ['agent.py'])
})
// env vars restored after block exits
```

### CI/CD: one-time deploy credential

```typescript
// In your release script
await sirr.push('DEPLOY_TOKEN', process.env.PERMANENT_DEPLOY_TOKEN!, { reads: 1 })
await execa('sirr', ['run', '--', './deploy.sh'])
// DEPLOY_TOKEN was read once by deploy.sh and is now deleted
```

---

## Related

- [SirrVault/sirr](https://github.com/SirrVault/sirr) — server
- [SirrVault/cli](https://github.com/SirrVault/cli) — CLI
- [SirrVault/python](https://github.com/SirrVault/python) — Python client
- [SirrVault/dotnet](https://github.com/SirrVault/dotnet) — .NET client
