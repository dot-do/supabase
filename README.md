# supabase.do

> Backend-as-a-Service. Edge-Native. AI-First. Natural Language.

Supabase charges for hosted Postgres, complex query builders, and infrastructure you don't control. Developers write `.from().select().eq()` chains that no one can read aloud.

**supabase.do** is the open-source alternative. Durable Objects + SQLite. Natural language queries. Deploys in seconds, not minutes.

## AI-Native API

```typescript
import { supabase } from 'supabase.do'           // Full SDK
import { supabase } from 'supabase.do/tiny'      // Minimal client
import { supabase } from 'supabase.do/realtime'  // Subscriptions only
```

Natural language for database operations:

```typescript
import { supabase } from 'supabase.do'

// Talk to it like a colleague
const tasks = await supabase`pending tasks sorted by due date`
const users = await supabase`users who signed up today`
const overdue = await supabase`tasks overdue assigned to ralph`

// Chain like sentences
await supabase`high priority bugs`
  .notify(`Bug needs attention`)

// Agents watching other agents
await supabase`watch tasks where assignee = ralph on insert`.notify(tom)
```

## The Problem

Supabase (the company) dominates the "Postgres-as-a-Service" market:

| What Supabase Charges | The Reality |
|-----------------------|-------------|
| **Pro Plan** | $25/month minimum |
| **Database Size** | $0.125/GB after 8GB |
| **Egress** | $0.09/GB after 50GB |
| **Realtime** | Limited connections on free tier |
| **Edge Functions** | Pay per invocation |
| **Vendor Lock-in** | Your data on their servers |

### The Query Builder Tax

Every Supabase tutorial teaches this:

```typescript
// This is what developers write today
const { data, error } = await supabase
  .from('tasks')
  .select('*')
  .eq('status', 'pending')
  .order('created_at', { ascending: false })
```

Try reading that aloud to a colleague. You can't. It's code for machines, not humans.

## The Solution

**supabase.do** reimagines BaaS for the AI era:

```
Supabase (hosted)              supabase.do
-----------------------------------------------------------------
$25/month minimum              $0 - run your own
Their Postgres                 Your Cloudflare account
Query builder chains           Natural language
Centralized database           Durable Object per agent
Manual scaling                 Infinite automatic scaling
Complex API                    Talk to it like a colleague
```

## One-Click Deploy

```bash
npx create-dotdo supabase
```

A complete backend. Running on infrastructure you control. Natural language from day one.

```typescript
import { Supabase } from 'supabase.do'

export default Supabase({
  name: 'my-startup',
  domain: 'db.my-startup.com',
})
```

## Features

### Querying Data

```typescript
// Find anything naturally
const tasks = await supabase`all tasks`
const pending = await supabase`pending tasks`
const urgent = await supabase`high priority bugs assigned to ralph`

// AI infers what you need
await supabase`tasks`                  // returns all tasks
await supabase`pending tasks`          // filters by status
await supabase`tasks created today`    // filters by date
```

### Inserting Data

```typescript
// Just say what you want to create
await supabase`new task: build authentication for ralph`
await supabase`add bug: login fails on mobile`

// Or with explicit data
await supabase`insert into tasks: ${{ title: 'Build auth', assignee: 'ralph' }}`
```

### Updating Data

```typescript
// Natural updates
await supabase`mark task 123 as complete`
await supabase`assign all pending tasks to tom`
await supabase`set priority high on bugs older than 7 days`
```

### Deleting Data

```typescript
// Natural deletes
await supabase`delete completed tasks older than 30 days`
await supabase`remove task 123`
```

### Real-time Subscriptions

```typescript
// Watch for changes naturally
await supabase`watch tasks on insert`.notify(tom)
await supabase`watch bugs where priority = high`.alert()

// Agent-to-agent watching
import { tom, ralph } from 'agents.do'

await supabase`watch ralph's builds on insert`
  .each(build => tom`review ${build}`)
```

### Auth

```typescript
// Sign in naturally
await supabase`sign in tom@agents.do`
await supabase`sign in with github`
await supabase`who am i`

// Agent authentication
const session = await supabase`authenticate tom with ${env.TOM_API_KEY}`
```

### Storage

```typescript
// File operations in plain English
await supabase`store attachments report.pdf`
await supabase`download attachments/report.pdf`
await supabase`url for attachments/report.pdf`

// Batch operations
await supabase`list files in attachments`
await supabase`delete attachments/old-report.pdf`
```

### Edge Functions

```typescript
// Invoke functions naturally
await supabase`run process-webhook with ${{ event: 'user.created' }}`

// Define functions with natural triggers
await supabase`on user created run send-welcome-email`
```

### Promise Pipelining

Work chains without `Promise.all`:

```typescript
// Find tasks, assign them, notify assignees - one round trip
const notified = await supabase`unassigned high priority tasks`
  .map(task => supabase`assign ${task} to available developer`)
  .map(assigned => assigned.assignee`you have a new task: ${assigned.title}`)

// Review chain
await supabase`pending code reviews`
  .map(pr => [tom, priya, quinn].map(r => r`review ${pr}`))
```

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

### Storage Tiers

| Tier | Storage | Use Case | Query Speed |
|------|---------|----------|-------------|
| **Hot** | SQLite | Active records, recent data | <10ms |
| **Warm** | R2 + SQLite Index | Historical data (2-7 years) | <100ms |
| **Cold** | R2 Archive | Compliance retention (7+ years) | <1s |

### Tiered Storage

```typescript
// Automatic tier selection - just query naturally
await supabase`recent logs`           // -> hot tier
await supabase`logs from last year`   // -> warm tier
await supabase`logs from 2020`        // -> cold tier, promoted on access
```

## Per-Agent Databases

Every AI agent gets their own isolated Supabase instance:

```typescript
import { tom, ralph, priya } from 'agents.do'

// Each agent has their own complete database
const tomDb = supabase.for(tom)
const ralphDb = supabase.for(ralph)
const priyaDb = supabase.for(priya)

// Natural queries scoped to each agent
await tomDb`my pending reviews`
await ralphDb`my active builds`
await priyaDb`roadmap items I own`
```

Not a shared database with row-level security. Not a multi-tenant nightmare. Each agent has their own complete instance.

## vs Supabase (Hosted)

| Feature | Supabase (Hosted) | supabase.do |
|---------|-------------------|-------------|
| **Minimum Cost** | $25/month | $0 - run your own |
| **Architecture** | Centralized Postgres | Edge-native Durable Objects |
| **Query API** | `.from().select().eq()` chains | Natural language |
| **Per-Agent DB** | Multi-tenant with RLS | Complete isolation |
| **Data Location** | Their servers | Your Cloudflare account |
| **Real-time** | Limited connections | Unlimited watchers |
| **Scaling** | Manual, pay more | Automatic, infinite |
| **Lock-in** | Their infrastructure | MIT licensed |

## Use Cases

### Task Management

```typescript
// Natural task operations
await supabase`create task: review PR 123 for tom`
await supabase`pending tasks sorted by priority`
await supabase`tasks overdue this week`

// Bulk operations
await supabase`close all completed tasks older than 30 days`
```

### Agent Memory

```typescript
// Each agent remembers their work
await supabase`store context: ${{ topic: 'auth', decisions: [...] }}`
await supabase`recall context about auth`
await supabase`what did I work on yesterday`
```

### Event Logs

```typescript
// Natural logging
await supabase`log: user 123 signed in from mobile`
await supabase`recent errors`
await supabase`activity for user 123 last 24 hours`
```

### Analytics

```typescript
// Query metrics naturally
await supabase`signups this month by country`
await supabase`daily active users trend`
await supabase`top 10 features by usage`
```

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
- Natural language API
- Compatible with the original when needed

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

// AI agents with full-stack infrastructure
const startup = {
  product: priya,      // specs, roadmaps
  engineering: ralph,  // builds features
  tech: tom,           // architecture, review
  marketing: mark,     // content, launches
}

// Each agent gets their own database, works naturally
await priya`create roadmap item: launch auth feature`
await ralph`log build: auth v1.0.0 deployed`
await tom`store review: PR 123 approved with comments`
await mark`draft announcement: auth is live`
```

Both kinds of workers. Working for you.

## Contributing

supabase.do is open source under the MIT license.

```bash
git clone https://github.com/dotdo/supabase.do
cd supabase.do
pnpm install
pnpm test
```

## License

MIT

---

<p align="center">
  <strong>Query builder chains end here.</strong>
  <br />
  Natural language. Edge-native. AI-first.
  <br /><br />
  <a href="https://supabase.do">Website</a> |
  <a href="https://docs.supabase.do">Docs</a> |
  <a href="https://discord.gg/dotdo">Discord</a> |
  <a href="https://github.com/dotdo/supabase.do">GitHub</a>
</p>
