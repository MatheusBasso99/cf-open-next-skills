# Self-Hosting: Cloudflare Workers via OpenNext

Deploy Next.js 16 to Cloudflare Workers using `@opennextjs/cloudflare`.
This stack uses the Node.js runtime (via `nodejs_compat`) — never the Edge runtime.

## Stack

- Next.js 16 + OpenNext v1.17.1
- `@opennextjs/cloudflare` adapter
- Wrangler >= 3.99.0
- pnpm (required package manager)

---

## New Project

```bash
pnpm create cloudflare@latest -- my-next-app --framework=next --platform=workers
```

---

## Existing Project: Manual Setup

### 1. Install dependencies

```bash
pnpm add @opennextjs/cloudflare@latest
pnpm add -D wrangler@latest
```

### 2. `wrangler.jsonc`

Create at the project root. The `name` value must be consistent — it is referenced by the `WORKER_SELF_REFERENCE` service binding and the Tag Cache.

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "main": ".open-next/worker.js",
  "name": "my-app",
  "compatibility_date": "2024-12-30",
  "compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
  "assets": {
    "directory": ".open-next/assets",
    "binding": "ASSETS"
  },
  "services": [
    {
      "binding": "WORKER_SELF_REFERENCE",
      "service": "my-app"
    }
  ],
  "r2_buckets": [
    {
      "binding": "NEXT_INC_CACHE_R2_BUCKET",
      "bucket_name": "<BUCKET_NAME>"
    }
  ],
  "durable_objects": {
    "bindings": [
      { "name": "NEXT_CACHE_DO_QUEUE", "class_name": "DOQueueHandler" },
      { "name": "NEXT_TAG_CACHE_DO_SHARDED", "class_name": "DOShardedTagCache" },
      { "name": "NEXT_CACHE_DO_PURGE", "class_name": "BucketCachePurge" }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["DOQueueHandler", "DOShardedTagCache", "BucketCachePurge"]
    }
  ],
  "images": {
    "binding": "IMAGES"
  }
}
```

For small sites replace the Durable Objects tag cache block with a D1 binding:

```jsonc
"d1_databases": [
  {
    "binding": "NEXT_TAG_CACHE_D1",
    "database_id": "<DATABASE_ID>",
    "database_name": "<DATABASE_NAME>"
  }
]
```

### 3. `open-next.config.ts`

Production baseline with R2 + regional cache + DO queue + sharded tag cache:

```ts
import { defineCloudflareConfig } from "@opennextjs/cloudflare";
import r2IncrementalCache from "@opennextjs/cloudflare/overrides/incremental-cache/r2-incremental-cache";
import { withRegionalCache } from "@opennextjs/cloudflare/overrides/incremental-cache/regional-cache";
import doQueue from "@opennextjs/cloudflare/overrides/queue/do-queue";
import doShardedTagCache from "@opennextjs/cloudflare/overrides/tag-cache/do-sharded-tag-cache";
import { purgeCache } from "@opennextjs/cloudflare/overrides/cache-purge/index";

export default defineCloudflareConfig({
  incrementalCache: withRegionalCache(r2IncrementalCache, {
    mode: "long-lived",
    bypassTagCacheOnCacheHit: true,
  }),
  queue: doQueue,
  tagCache: doShardedTagCache({ baseShardSize: 12 }),
  enableCacheInterception: true,
  cachePurge: purgeCache({ type: "direct" }),
});
```

For small sites swap `doShardedTagCache` for `d1NextTagCache`:

```ts
import d1NextTagCache from "@opennextjs/cloudflare/overrides/tag-cache/d1-next-tag-cache";

export default defineCloudflareConfig({
  incrementalCache: r2IncrementalCache,
  queue: doQueue,
  tagCache: d1NextTagCache,
});
```

For SSG-only sites (no revalidation):

```ts
import staticAssetsIncrementalCache from "@opennextjs/cloudflare/overrides/incremental-cache/static-assets-incremental-cache";

export default defineCloudflareConfig({
  incrementalCache: staticAssetsIncrementalCache,
  enableCacheInterception: true,
});
```

### 4. `next.config.ts`

Add the dev integration call **outside** the exported config object so it only runs during `next dev`:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {};

export default nextConfig;

import { initOpenNextCloudflareForDev } from "@opennextjs/cloudflare";
initOpenNextCloudflareForDev();
```

Never set `export const runtime = "edge"` anywhere in the codebase. The Node.js runtime is the only supported target.

### 5. `.dev.vars`

```
NEXTJS_ENV=development
```

### 6. `package.json` scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "preview": "opennextjs-cloudflare build && opennextjs-cloudflare preview",
    "deploy": "opennextjs-cloudflare build && opennextjs-cloudflare deploy",
    "upload": "opennextjs-cloudflare build && opennextjs-cloudflare upload",
    "cf-typegen": "wrangler types --env-interface CloudflareEnv cloudflare-env.d.ts"
  }
}
```

### 7. Static asset caching

Create `public/_headers`:

```
/_next/static/*
  Cache-Control: public,max-age=31536000,immutable
```

### 8. `.gitignore`

```
.open-next
```

---

## Cloudflare Resources Setup

### R2 bucket

```bash
pnpx wrangler@latest r2 bucket create <YOUR_BUCKET_NAME>
```

### D1 database (small-site tag cache only)

```bash
pnpx wrangler@latest d1 create <DATABASE_NAME>
```

Create the required table:

```sql
CREATE TABLE IF NOT EXISTS revalidations (
  tag TEXT NOT NULL,
  revalidatedAt INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_tag ON revalidations (tag);
```

---

## Environment Variables (Secrets)

Required for automatic cache purge on custom domains:

```bash
pnpx wrangler@latest secret put CACHE_PURGE_API_TOKEN
pnpx wrangler@latest secret put CACHE_PURGE_ZONE_ID
```

`CACHE_PURGE_API_TOKEN` must have the `Cache Purge` permission.
`CACHE_PURGE_ZONE_ID` is the zone ID of your deployment domain.

All other env vars live in `.dev.vars` (local) or are set via `wrangler secret put` (production).
Access them exclusively through `lib/env.ts` — never via `process.env` directly in application code.

---

## Local Development

```bash
pnpm dev
```

Uses Next.js dev server with Cloudflare bindings available via `initOpenNextCloudflareForDev()`.

To preview the app running in the actual Workers runtime:

```bash
pnpm preview
```

---

## Deploy

```bash
pnpm deploy
```

To upload a new version without promoting it immediately:

```bash
pnpm upload
```

Or connect a GitHub/GitLab repository and let Cloudflare CI/CD build and deploy on each merge to the production branch.

---

## Architecture Constraints for This Stack

### No double Worker invocations

RSC resolves data in a single Worker execution. Internal Route Handlers called by the Next.js app itself add a second invocation for the same page load and must not be used for data fetching. Use Server Components and Server Actions instead.

### Token security

`lib/api/client.ts` is the only layer that touches tokens. Tokens are injected server-side via httpOnly cookies. No token or Authorization header is ever passed to client-side code.

### Runtime

`nodejs_compat` is enabled in `wrangler.jsonc`. All services under `lib/api/services/` run with full Node.js APIs — no Edge Runtime restrictions.

### Route Handlers

Use Route Handlers only for external callers: webhooks (Stripe, etc.), OAuth callbacks, file proxies, and health check endpoints. Never use them to feed Zustand stores or replace Server Actions for mutations from this app.

---

## Caching Strategy Reference

| Site profile | Incremental Cache | Queue | Tag Cache |
|---|---|---|---|
| SSG only | `staticAssetsIncrementalCache` | none | none |
| Small + revalidation | R2 | DO Queue | D1 (`d1NextTagCache`) |
| Large + revalidation | R2 + regional cache | DO Queue | `doShardedTagCache` |
| Staging / low-traffic | R2 | memory queue | D1 |

`enableCacheInterception: true` improves cold-start performance for ISR/SSG routes. Disable it when using PPR.

---

## Feature Support Matrix

| Feature | Supported | Notes |
|---|---|---|
| App Router | Yes | Primary target |
| Server Components (RSC) | Yes | Single Worker execution |
| Server Actions | Yes | POST only, no HTTP caching |
| Route Handlers | Yes | External callers only |
| SSG | Yes | |
| SSR | Yes | Works without caching config |
| ISR | Yes | Requires R2 + Queue setup |
| PPR | Yes | Disable `enableCacheInterception` |
| `use cache` | Yes | Next.js 16 Cache Components |
| Middleware (`proxy.ts`) | Yes | Node.js middleware (v15.2+ only) |
| Node Middleware | No | Not yet supported |
| Edge runtime | No | Remove any `runtime = "edge"` exports |
| Image optimization | Yes | Configure Cloudflare Images binding |
| `after()` | Yes | |
| Turbopack | Yes | |

---

## Pre-Deployment Checklist

1. No `export const runtime = "edge"` anywhere in source files
2. R2 bucket created and bound as `NEXT_INC_CACHE_R2_BUCKET`
3. `WORKER_SELF_REFERENCE` service binding name matches `"name"` in `wrangler.jsonc`
4. `public/_headers` includes immutable cache rule for `/_next/static/*`
5. `.open-next` added to `.gitignore`
6. Secrets set via `wrangler secret put` for production
7. `pnpm build` passes locally before deploying
8. `pnpm preview` tested locally in Workers runtime
9. All env vars accessed through `lib/env.ts`, never `process.env` directly

---

## Debugging

Add to `.env`:

```
NEXT_PRIVATE_DEBUG_CACHE=1
```

This outputs logs whenever the cache is accessed, from both Next.js and the cache adapter.
