# supabase.do

Supabase on Cloudflare Durable Objects - A complete backend for every AI agent.

## The Problem

AI agents need databases. Millions of them. Running in parallel. Each isolated. Each with their own state.

Traditional databases were built for humans:
- One shared database for many users
- Centralized infrastructure
- Manual scaling
- Expensive per-instance

AI agents need the opposite:
- One database per agent
- Distributed by default
- Infinite automatic scaling
- Free at the instance level, pay for usage

## The Vision

Every AI agent gets their own Supabase.

```typescript
import { tom, ralph, priya } from 'agents.do'
import { Supabase } from 'supabase.do'

// Each agent has their own isolated database
const tomDb = Supabase.for(tom)
const ralphDb = Supabase.for(ralph)
const priyaDb = Supabase.for(priya)

// Full Supabase API
await tomDb.from('reviews').insert({ pr: 123, status: 'approved' })
await ralphDb.from('builds').select('*').eq('status', 'pending')
await priyaDb.from('roadmap').update({ priority: 1 }).eq('feature', 'auth')
```

Not a shared database with row-level security. Not a multi-tenant nightmare. Each agent has their own complete Supabase instance.

## Features

- **Postgres-like SQL** - SQLite under the hood, Postgres API on top
- **Real-time Subscriptions** - Agents watching for changes from other agents
- **Auth** - Every agent has identity (keys, tokens, sessions)
- **Storage** - File attachments backed by R2
- **Edge Functions** - Agent-defined serverless functions
- **Tiered Storage** - Hot (SQLite) / Warm (R2) / Cold (Archive)
- **MCP Tools** - Model Context Protocol for AI-native database operations

## Architecture

```
                    +-----------------------+
                    |     supabase.do       |
                    |   (Cloudflare Worker) |
                    +-----------------------+
                              |
              +---------------+---------------+
              |               |               |
    +------------------+ +------------------+ +------------------+
    | SupabaseDO (Tom) | | SupabaseDO (Rae) | | SupabaseDO (...) |
    |   SQLite + R2    | |   SQLite + R2    | |   SQLite + R2    |
    +------------------+ +------------------+ +------------------+
              |               |               |
              +---------------+---------------+
                              |
                    +-------------------+
                    |        R2         |
                    | (warm/cold tier)  |
                    +-------------------+
```

**Key insight**: Durable Objects provide single-threaded, strongly consistent state. Each agent's database is a Durable Object. SQLite handles queries. R2 handles large objects and cold storage.

**Scale**: Millions of concurrent isolated instances. Each is free to create. You pay for what you use.

## Installation

```bash
npm install supabase.do
```

## Quick Start

### Basic Operations

```typescript
import { Supabase } from 'supabase.do'

const db = new Supabase(env.SUPABASE)

// Insert
await db.from('tasks').insert({
  title: 'Build authentication',
  status: 'pending',
  assignee: 'ralph'
})

// Select with filters
const tasks = await db
  .from('tasks')
  .select('*')
  .eq('status', 'pending')
  .order('created_at', { ascending: false })

// Update
await db
  .from('tasks')
  .update({ status: 'complete' })
  .eq('id', task.id)

// Delete
await db.from('tasks').delete().eq('id', task.id)
```

### Real-time Subscriptions

```typescript
import { Supabase } from 'supabase.do'

const db = new Supabase(env.SUPABASE)

// Subscribe to changes
const subscription = db
  .channel('tasks')
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'tasks',
    filter: 'assignee=eq.ralph'
  }, (payload) => {
    console.log('New task for Ralph:', payload.new)
  })
  .subscribe()

// Agents watching other agents
import { tom, ralph } from 'agents.do'

tom.watch(ralph.db.from('builds'), {
  event: 'INSERT'
}, async (build) => {
  await tom`review ${build}`
})
```

### Auth

```typescript
import { Supabase } from 'supabase.do'

const db = new Supabase(env.SUPABASE)

// Agent authentication
const { data: { session } } = await db.auth.signIn({
  agent: 'tom@agents.do',
  key: env.TOM_API_KEY
})

// Human authentication
const { data: { url } } = await db.auth.signInWithOAuth({
  provider: 'github'
})

// JWT validation
const { data: { user } } = await db.auth.getUser(jwt)
```

### Storage

```typescript
import { Supabase } from 'supabase.do'

const db = new Supabase(env.SUPABASE)

// Upload file
const { data, error } = await db.storage
  .from('attachments')
  .upload('report.pdf', file)

// Download file
const { data: blob } = await db.storage
  .from('attachments')
  .download('report.pdf')

// Get public URL
const { data: { publicUrl } } = db.storage
  .from('attachments')
  .getPublicUrl('report.pdf')
```

### Edge Functions

```typescript
import { Supabase } from 'supabase.do'

const db = new Supabase(env.SUPABASE)

// Invoke an edge function
const { data, error } = await db.functions.invoke('process-webhook', {
  body: { event: 'user.created', data: user }
})

// Define an edge function
export const processWebhook = db.functions.define('process-webhook', async (req) => {
  const { event, data } = await req.json()
  // Handle webhook...
  return { processed: true }
})
```

### MCP Tools

```typescript
import { supabaseTools, invokeTool } from 'supabase.do/mcp'

// List available database tools
console.log(supabaseTools.map(t => t.name))
// ['db_query', 'db_insert', 'db_update', 'db_delete', 'db_subscribe', ...]

// Invoke a tool
const result = await invokeTool('db_query', {
  table: 'tasks',
  select: '*',
  filter: { status: 'pending' }
})

// AI-native database access
await invokeTool('db_query', {
  natural: 'find all tasks assigned to Ralph that are overdue'
})
```

### Durable Object

```typescript
import { SupabaseDO } from 'supabase.do/do'

// In your worker
export { SupabaseDO }

export default {
  async fetch(request, env) {
    // Each agent gets their own Supabase instance
    const id = env.SUPABASE.idFromName('agent-tom')
    const stub = env.SUPABASE.get(id)
    return stub.fetch(request)
  }
}
```

### Tiered Storage

```typescript
import { TieredSupabase } from 'supabase.do/storage'

const db = new TieredSupabase({
  hot: env.SUPABASE,      // Durable Object (fast, small)
  warm: env.R2_BUCKET,    // R2 (large blobs)
  cold: env.ARCHIVE,      // Archive (infrequent)
  thresholds: {
    hotMaxRows: 100000,           // 100k rows
    hotMaxSize: 10 * 1024 * 1024, // 10MB
  }
})

// Automatic tier selection
await db.from('logs').insert(smallRecord)    // -> hot tier
await db.from('logs').insert(hugeRecord)     // -> warm tier

// Cold queries with auto-promotion
const archived = await db.from('logs')
  .select('*')
  .lt('created_at', '2024-01-01')  // -> cold tier, promoted on access
```

## API Overview

### Database (`supabase.do`)

**Query Builder**
- `from(table)` - Start a query
- `select(columns)` - Select columns
- `insert(data)` - Insert rows
- `update(data)` - Update rows
- `delete()` - Delete rows
- `upsert(data)` - Insert or update

**Filters**
- `eq(column, value)` - Equal
- `neq(column, value)` - Not equal
- `gt(column, value)` - Greater than
- `lt(column, value)` - Less than
- `like(column, pattern)` - LIKE match
- `in(column, values)` - IN list
- `is(column, value)` - IS NULL/NOT NULL
- `or(filters)` - OR conditions
- `not(filter)` - NOT condition

**Ordering & Pagination**
- `order(column, options)` - Sort results
- `limit(count)` - Limit rows
- `range(from, to)` - Pagination

### Real-time (`supabase.do/realtime`)

- `channel(name)` - Create channel
- `on(event, filter, callback)` - Subscribe to events
- `subscribe()` - Start subscription
- `unsubscribe()` - Stop subscription

### Auth (`supabase.do/auth`)

- `signIn(credentials)` - Sign in
- `signOut()` - Sign out
- `getSession()` - Get current session
- `getUser()` - Get current user
- `onAuthStateChange(callback)` - Auth state listener

### Storage (`supabase.do/storage`)

- `from(bucket)` - Select bucket
- `upload(path, file)` - Upload file
- `download(path)` - Download file
- `remove(paths)` - Delete files
- `list(path)` - List files
- `getPublicUrl(path)` - Get URL

### Functions (`supabase.do/functions`)

- `invoke(name, options)` - Call function
- `define(name, handler)` - Create function

### MCP Tools (`supabase.do/mcp`)

- `supabaseTools` - Available tool definitions
- `invokeTool(name, params)` - Execute a tool
- `registerTool(tool)` - Add custom tool

## The Rewrites Ecosystem

supabase.do is part of the rewrites family - reimplementations of popular infrastructure on Cloudflare Durable Objects:

| Rewrite | Original | Purpose |
|---------|----------|---------|
| [fsx.do](https://fsx.do) | fs (Node.js) | Filesystem for AI |
| [gitx.do](https://gitx.do) | git | Version control for AI |
| **supabase.do** | Supabase | Postgres/BaaS for AI |
| mongo.do | MongoDB | Document database for AI |
| kafka.do | Kafka | Event streaming for AI |
| nats.do | NATS | Messaging for AI |
| redis.do | Redis | Cache/pub-sub for AI |
| neo4j.do | Neo4j | Graph database for AI |

Each rewrite follows the same pattern:
- Durable Object per instance (per agent)
- SQLite for hot tier storage
- R2 for warm/cold tier storage
- MCP tools for AI-native access
- Compatible API with the original

## Why Durable Objects?

1. **Single-threaded consistency** - No race conditions, no distributed locks
2. **Per-instance isolation** - Each agent's data is completely separate
3. **Automatic scaling** - Millions of instances, zero configuration
4. **Global distribution** - Data lives at the edge
5. **SQLite inside** - Real SQL, real transactions, real performance
6. **R2 integration** - Unlimited storage for large objects

## The workers.do Platform

supabase.do is a core service of [workers.do](https://workers.do) - the platform for building Autonomous Startups.

```typescript
import { priya, ralph, tom, mark } from 'agents.do'
import { Supabase } from 'supabase.do'

// AI agents with full-stack infrastructure
const startup = {
  product: priya,      // specs, roadmaps
  engineering: ralph,  // builds features
  tech: tom,           // architecture, review
  marketing: mark,     // content, launches
}

// Each agent has their own database, filesystem, git repo
for (const [role, agent] of Object.entries(startup)) {
  const db = Supabase.for(agent)
  await db.from('context').insert({
    role,
    started: new Date(),
    status: 'active'
  })
}
```

Both kinds of workers. Working for you.

## License

MIT
