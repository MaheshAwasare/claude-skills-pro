---
name: cloudflare-workers
description: Build production apps on Cloudflare Workers — Workers + KV + D1 + R2 + Durable Objects + Queues, wrangler config, bindings, the runtime constraints (no Node APIs, no long-running compute), and the patterns that exploit edge architecture instead of fighting it. Use when building a new edge-deployed service or migrating from Lambda for cost.
---

# Cloudflare Workers

Workers run JS/TS at 300+ edge locations with sub-millisecond cold starts. Trade-off: a constrained runtime (no Node APIs, 128MB RAM, 30s CPU on paid plan, 50ms on free). The right shape: thin compute that fans out to bindings (KV, D1, R2, DO, Queues).

## When to use

- Stateless API at the edge — auth proxy, GraphQL gateway, image transforms.
- Static + dynamic hybrid — `Pages` for static, Worker for dynamic routes.
- Real-time at scale — Durable Objects for per-room state.
- Cost-driven migration from Lambda + DynamoDB.

## When NOT to use

- Heavy CPU work (ML inference, video processing) — buy compute elsewhere.
- Long-running connections (>30s WebSocket from a regular Worker) — use DO with hibernating WebSockets instead.
- Apps that need full Node ecosystem (e.g. Puppeteer) — different runtime.

## The bindings model (key insight)

Workers don't talk to "AWS S3 over the network." They have *bindings* — references to resources injected at runtime by Cloudflare. No credentials in code, no SDK overhead.

| Binding | What it gives you |
|---|---|
| KV | Eventually-consistent global key-value store. ~50ms reads, slow writes. |
| D1 | SQLite at the edge. ms reads in same region, replicated. |
| R2 | S3-compatible blob storage, **no egress fees**. |
| Durable Objects | Single-instance, single-threaded actors. Use for per-entity state. |
| Queues | At-least-once message queue. Producers + consumers in Workers. |
| Hyperdrive | Pooled connection to your existing Postgres/MySQL. |

## Wrangler config

```toml
# wrangler.toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2026-05-01"
compatibility_flags = ["nodejs_compat"]   # opts in to a subset of Node APIs

[[kv_namespaces]]
binding = "CACHE"
id = "abc123..."

[[d1_databases]]
binding = "DB"
database_name = "myapp-prod"
database_id = "def456..."

[[r2_buckets]]
binding = "ASSETS"
bucket_name = "myapp-uploads"

[[durable_objects.bindings]]
name = "ROOMS"
class_name = "Room"

[[migrations]]
tag = "v1"
new_classes = ["Room"]

[vars]
APP_URL = "https://example.com"

[env.staging]
vars = { APP_URL = "https://staging.example.com" }
```

```ts
// Worker handler — bindings appear as `env.X`
export interface Env {
  CACHE: KVNamespace;
  DB: D1Database;
  ASSETS: R2Bucket;
  ROOMS: DurableObjectNamespace;
  APP_URL: string;
}

export default {
  async fetch(req: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(req.url);
    if (url.pathname === "/healthz") return new Response("ok");
    return handleApi(req, env, ctx);
  },
};
```

## D1 (SQLite at the edge)

```ts
const { results } = await env.DB.prepare("SELECT * FROM users WHERE id = ?").bind(userId).all();

// Batch (transactional)
const stmts = [
  env.DB.prepare("INSERT INTO users (id, email) VALUES (?, ?)").bind(id, email),
  env.DB.prepare("INSERT INTO audit (user_id, action) VALUES (?, ?)").bind(id, "signup"),
];
await env.DB.batch(stmts);
```

Migrations via `wrangler d1 migrations`:
```bash
wrangler d1 migrations create myapp-prod add_users
# edit migrations/0001_add_users.sql
wrangler d1 migrations apply myapp-prod --local       # apply locally
wrangler d1 migrations apply myapp-prod --remote      # apply to prod
```

D1 is **read-replicated globally; writes go to the primary**. Same-region reads are <10ms; cross-region writes can be 100ms+. Design your access pattern around this.

## KV (cache, not DB)

```ts
// Read with stale-while-revalidate
const cached = await env.CACHE.get(`user:${id}`, { cacheTtl: 60, type: "json" });
if (cached) return Response.json(cached);

// Compute fresh
const fresh = await fetchFromOrigin();
ctx.waitUntil(env.CACHE.put(`user:${id}`, JSON.stringify(fresh), { expirationTtl: 300 }));
return Response.json(fresh);
```

KV is **eventually consistent** — writes propagate over ~60s globally. Don't use it as a primary store; use it as a cache.

## R2 (S3 without egress fees)

```ts
// Upload
await env.ASSETS.put(`avatars/${userId}.png`, file.body, {
  httpMetadata: { contentType: file.type },
  customMetadata: { uploaderId: userId },
});

// Read
const obj = await env.ASSETS.get(`avatars/${userId}.png`);
if (!obj) return new Response("not found", { status: 404 });
return new Response(obj.body, { headers: { "Content-Type": obj.httpMetadata?.contentType ?? "" } });
```

R2 charges $0 for egress — vs S3's $0.09/GB. For media-heavy apps, this is the entire reason to use Cloudflare.

## Durable Objects (per-entity actors)

```ts
export class Room {
  state: DurableObjectState;
  sessions = new Set<WebSocket>();

  constructor(state: DurableObjectState) { this.state = state; }

  async fetch(req: Request): Promise<Response> {
    if (req.headers.get("upgrade") === "websocket") {
      const pair = new WebSocketPair();
      const [client, server] = Object.values(pair);
      server.accept();
      this.sessions.add(server);
      server.addEventListener("message", (e) => this.broadcast(e.data));
      server.addEventListener("close", () => this.sessions.delete(server));
      return new Response(null, { status: 101, webSocket: client });
    }
    return new Response("ok");
  }

  broadcast(msg: any) {
    for (const ws of this.sessions) ws.send(typeof msg === "string" ? msg : JSON.stringify(msg));
  }
}
```

Each room (e.g. `room:abc`) gets exactly one Durable Object instance. State persists in DO storage. Perfect for: chat rooms, multiplayer games, collaborative editors, rate limiters.

For high-WS-count rooms, use **hibernating WebSockets** — DO sleeps when idle, wakes on message, doesn't bill while idle.

## Queues (at-least-once)

```toml
# Producer
[[queues.producers]]
binding = "EMAIL_QUEUE"
queue   = "emails"

# Consumer (separate Worker)
[[queues.consumers]]
queue = "emails"
max_batch_size = 10
max_batch_timeout = 30
```

```ts
// Producer
await env.EMAIL_QUEUE.send({ to: "x@y.com", template: 7, params: { code: "1234" } });

// Consumer Worker
export default {
  async queue(batch: MessageBatch<EmailMsg>, env: Env) {
    for (const msg of batch.messages) {
      try {
        await sendBrevo(msg.body);
        msg.ack();
      } catch {
        msg.retry({ delaySeconds: 60 });
      }
    }
  },
};
```

Always `ack()` or `retry()` per message. Unacked messages re-deliver. Dead-letter queue config in `wrangler.toml` for messages that fail too many times.

## Anti-patterns

- **Long-running compute** — Workers cap at 30s CPU (paid). For longer, queue + multiple Workers.
- **Reading `process.env`** — Workers don't expose `process`. Use `env.VAR` (binding) or `wrangler secret`.
- **Importing Node-only packages** — most libraries break. Check `nodejs_compat` flag and the supported subset.
- **KV as primary store** — eventually consistent. Lost writes feel like data corruption.
- **Hardcoding region-specific assumptions** — Workers run in 300+ POPs; assume nothing about location.
- **Per-request `fetch` to a third-party API without timeouts** — slow APIs blow your subrequest budget (50/req on free).
- **Mutating module-scope state** — Workers reuse isolates across requests; mutable global state leaks across users.
- **Not using `ctx.waitUntil` for fire-and-forget** — without it, your bg work is killed when the response is sent.
- **One Worker for everything** — separate per-domain concerns; small Workers are easier to reason about.
- **Storing user uploads in KV** — KV has 25MB/value cap and pricing curves. R2 is the answer.
- **Writing to D1 from many regions concurrently expecting Postgres semantics** — D1 has different consistency. Design around it.

## Verify it worked

- [ ] `wrangler dev` runs the Worker locally with bindings working (KV/D1/R2 all simulated).
- [ ] `wrangler deploy` deploys to prod; URL responds in < 100ms globally.
- [ ] Static asset upload to R2 returns successfully; download works; egress is $0.
- [ ] D1 migrations applied via `wrangler d1 migrations apply --remote`; queries work.
- [ ] KV cache: first request slow (origin), second fast (KV).
- [ ] DO room: two WebSocket clients send/receive each other's messages in real time.
- [ ] Queue: producer in Worker A sends; consumer in Worker B processes within seconds.
- [ ] `ctx.waitUntil` keeps a logging fetch alive after `Response` is sent.
- [ ] Secrets are set via `wrangler secret put NAME`, not in `wrangler.toml`.
- [ ] Staging env separate (separate D1, KV, R2 buckets).
