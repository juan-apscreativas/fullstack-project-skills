---
name: nextjs-cloudflare-deployment
description: >
  This skill should be used when the user asks to "deploy Next.js to Cloudflare", "set up OpenNext adapter", "configure Cloudflare Workers for Next.js", "fix next/image on Cloudflare", "configure wrangler for Next.js", "handle Cloudflare deployment issues", "set up KV or D1 cache for ISR", or "use @opennextjs/cloudflare".
---

# Next.js 15 on Cloudflare Workers (OpenNext Adapter)

Expert in deploying Next.js 15 applications to Cloudflare Workers via `@opennextjs/cloudflare`. This skill replaces the standard Docker/standalone deployment path described in `nextjs-feature-based` — the two approaches are mutually exclusive for a given environment. Docker standalone remains valid for other hosting targets (VPS, Railway, Fly.io, etc.).

> **Version constraint:** `@opennextjs/cloudflare` officially supports **Next.js 14 and 15**. Next.js 16 on Cloudflare Workers is not yet supported — stay on Next.js 15.x until the adapter is updated. Do not upgrade Next.js versions without first verifying adapter compatibility at https://github.com/opennextjs/opennextjs-cloudflare.

> **Documentation principle:** `docs/ADR/` must record the decision to use Cloudflare Workers as the deployment target and document any workarounds applied. Update the project CLAUDE.md to reflect that deployment uses OpenNext, not Docker standalone.

## Core Principles

1. **OpenNext is the build pipeline** — never use `output: 'standalone'` or `next build` alone for Cloudflare; use `opennextjs-cloudflare build`.
2. **Node.js runtime only** — never set `export const runtime = 'edge'` in any route or middleware; the OpenNext adapter runs on the Cloudflare Workers Node.js compat layer, not the Edge runtime.
3. **`next/image` requires an external loader** — the built-in Image Optimization server needs a Node.js process Cloudflare Workers doesn't provide; configure a loader (Cloudflare Images, Imgix, or a custom CDN) instead.
4. **`use cache` / `cacheLife` / `cacheTag` are Vercel-only** — do not use these Next.js cache directives; use `fetch` cache options or OpenNext's KV/R2/D1 incremental cache instead.
5. **Cloudflare bindings are accessed via `getCloudflareContext()`** — not via Node.js `process.env` for KV/R2/D1.
6. **Wrangler is the local preview tool** — use `wrangler dev` after `opennextjs-cloudflare build` to test locally; `npm run dev` continues to work for standard Next.js development.
7. **`output: 'standalone'` is removed** — it conflicts with the OpenNext build output. Keep it only in a separate Docker-targeted config if you maintain dual deployment.
8. **Document all workarounds in ADR** — every Cloudflare-specific deviation from the standard Next.js pattern needs an ADR entry.

## Incompatibility Map

| Next.js Feature | Cloudflare Status | Workaround |
|-----------------|-------------------|------------|
| `next/image` built-in optimization | ❌ Not supported | Use `loader` prop pointing to Cloudflare Images or external CDN |
| `export const runtime = 'edge'` | ❌ Not supported with OpenNext | Remove — Node.js compat is used instead |
| `use cache` / `cacheLife` / `cacheTag` | ❌ Vercel-only | Use `fetch` cache options + KV/R2 incremental cache |
| `output: 'standalone'` | ❌ Conflicts | Remove from `next.config.ts` when targeting Cloudflare |
| Multi-instance cache handlers (Docker) | ❌ Not applicable | Use `kvIncrementalCache` or `r2IncrementalCache` |
| `next start` / `node server.js` | ❌ Not applicable | Deployment via `wrangler deploy` |
| ISR / `revalidate` | ✅ Supported via KV + D1 tag cache | Requires `kvIncrementalCache` + `d1NextTagCache` in `open-next.config.ts` |
| Server Components + data fetching | ✅ Fully supported | No changes needed |
| API Routes / Route Handlers | ✅ Fully supported | No changes needed |
| Middleware (Proxy) | ✅ Supported | Runs as a Worker; avoid Node.js-only APIs |
| Streaming / Suspense | ✅ Supported | No changes needed |
| Parallel Routes / Intercepting Routes | ✅ Supported | No changes needed |
| PPR (Partial Prerendering) | ⚠️ Experimental | Set `enableCacheInterception: false` in OpenNext config |

## Setup

### 1 — Install the adapter
```bash
npm install --save-dev @opennextjs/cloudflare wrangler
```

### 2 — Update `next.config.ts`
Remove `output: 'standalone'` — OpenNext manages its own output. Keep all other Next.js config as-is.

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // output: 'standalone' — REMOVED for Cloudflare target
  images: {
    loader: 'custom',
    loaderFile: './src/shared/lib/image-loader.ts', // see next/image section below
  },
}

export default nextConfig
```

### 3 — Create `open-next.config.ts`
```typescript
// open-next.config.ts (project root)
import { defineCloudflareConfig } from '@opennextjs/cloudflare'
import { kvIncrementalCache } from '@opennextjs/cloudflare/kv-cache'
import { d1NextTagCache } from '@opennextjs/cloudflare/d1-cache'

export default defineCloudflareConfig({
  incrementalCache: kvIncrementalCache,   // KV for ISR page cache
  tagCache: d1NextTagCache,               // D1 for revalidation tags
  queue: 'direct',                        // or doQueue for high-traffic
})
```

For simpler projects without ISR, use the dummy (in-memory) implementations:
```typescript
export default defineCloudflareConfig() // all defaults — dummy caches
```

### 4 — Create `wrangler.jsonc`
```jsonc
// wrangler.jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-next-app",
  "main": ".open-next/worker.js",
  "compatibility_date": "2024-12-30",
  "compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
  "assets": {
    "directory": ".open-next/assets",
    "binding": "ASSETS"
  },
  // Add these only if using kvIncrementalCache:
  "kv_namespaces": [
    { "binding": "NEXT_INC_CACHE_KV", "id": "<your-kv-id>" }
  ],
  // Add these only if using d1NextTagCache:
  "d1_databases": [
    { "binding": "NEXT_TAG_CACHE_D1", "database_name": "next-tags", "database_id": "<your-d1-id>" }
  ],
  "vars": {
    "NEXTJS_ENV": "production"
  }
}
```

### 5 — Add build scripts to `package.json`
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "cf:build": "opennextjs-cloudflare build",
    "cf:preview": "npm run cf:build && wrangler dev",
    "cf:deploy": "npm run cf:build && opennextjs-cloudflare deploy"
  }
}
```

## next/image Workaround

The built-in Next.js image optimizer requires a running Node.js server — unavailable on Cloudflare Workers. Create a custom loader:

```typescript
// src/shared/lib/image-loader.ts
export default function cloudflareImageLoader({
  src,
  width,
  quality,
}: {
  src: string
  width: number
  quality?: number
}) {
  // Option A — Cloudflare Images (if enabled on your account)
  return `https://imagedelivery.net/<account-hash>/${src}/w=${width},q=${quality ?? 75}`

  // Option B — pass through to a CDN or your own resize service
  // return `https://cdn.example.com/resize?url=${src}&w=${width}&q=${quality ?? 75}`
}
```

Use `<Image>` components normally — the loader handles the rest:
```tsx
import Image from 'next/image'

<Image src="product-123" width={400} height={300} alt="Product" />
```

## Accessing Cloudflare Bindings

In Server Components, Route Handlers, and Middleware, use `getCloudflareContext()` to access KV, R2, D1, and request metadata:

```typescript
import { getCloudflareContext } from '@opennextjs/cloudflare'

export async function GET() {
  const { env, cf, ctx } = getCloudflareContext()

  // KV read
  const value = await env.MY_KV.get('key')

  // Geolocation from the request
  const country = cf?.country

  // Background task (doesn't block response)
  ctx.waitUntil(logAnalytics(country))

  return Response.json({ value, country })
}
```

## Local Development vs. Preview

| Mode | Command | Runtime |
|------|---------|---------|
| Development (fast) | `npm run dev` | Node.js — standard Next.js |
| Cloudflare preview | `npm run cf:preview` | Wrangler — simulates Workers runtime |
| Deploy to production | `npm run cf:deploy` | Cloudflare Workers |

Always test with `cf:preview` before deploying — behavior can differ from `npm run dev` because Workers uses V8 isolates with the Node.js compat layer, not a full Node.js runtime.

## Patterns

### Caching Strategy Decision Tree
```
Does the project use ISR (revalidate)?
├── No  → defineCloudflareConfig() with dummy caches (simplest)
└── Yes → kvIncrementalCache + d1NextTagCache
          ├── Low traffic → queue: 'direct'
          └── High traffic → queue: doQueue (Durable Objects)
```

### Environment Variables
Server-side env vars work normally via `process.env` for plain values. For Cloudflare bindings (KV, R2, D1) always use `getCloudflareContext().env` — they are not available in `process.env`.

## Anti-Patterns

### ❌ Edge Runtime in Route Files
```typescript
// WRONG — incompatible with OpenNext adapter
export const runtime = 'edge'
```

### ❌ Vercel Cache Directives
```typescript
// WRONG — Vercel-proprietary, silently ignored or breaks on Cloudflare
'use cache'
export const cacheLife = 'days'
```

### ❌ `output: 'standalone'` Left In
```typescript
// WRONG — conflicts with OpenNext build pipeline
const nextConfig = { output: 'standalone' }
```

### ❌ Running `next build` for Cloudflare
```bash
# WRONG — produces the wrong output format
npm run build && node .next/standalone/server.js

# CORRECT
npm run cf:build && wrangler dev   # preview
npm run cf:deploy                  # production
```

### ❌ Upgrading to Next.js 16 Without Checking Adapter
```bash
# WRONG — adapter does not yet support Next.js 16
npm install next@16

# CORRECT — verify support first, stay on 15.x until confirmed
```

## ADR Template for This Decision

Create `docs/ADR/NNN-cloudflare-workers-deployment.md`:

```markdown
# NNN — Deploy to Cloudflare Workers via OpenNext

## Status
Accepted

## Context
The project requires deployment to Cloudflare Workers instead of a traditional
Node.js host. The official Next.js `output: 'standalone'` + Docker path is not
compatible with the Cloudflare Workers runtime.

## Decision
Use `@opennextjs/cloudflare` as the build adapter. Next.js version is pinned to
15.x until the adapter officially supports 16.x.

## Consequences
- `output: 'standalone'` removed from next.config.ts
- `next/image` uses a custom loader (Cloudflare Images / external CDN)
- `export const runtime = 'edge'` is prohibited in all route files
- Vercel-specific cache directives (`use cache`, `cacheLife`, `cacheTag`) are prohibited
- Local preview requires `wrangler dev` after `opennextjs-cloudflare build`
- ISR caching uses KV (pages) + D1 (tags) instead of the filesystem
```

## Related Skills

Works well with: `nextjs-feature-based`, `frontend-feature-dev`, `api-contract-sync`

Replaces (for Cloudflare targets): the Docker/standalone deployment sections of `nextjs-feature-based`

## When to Use

This skill applies when the Next.js 15 application must be deployed to Cloudflare Workers. It does not apply to other hosting providers (Docker, Railway, Fly.io, Render) — for those, the standard `output: 'standalone'` + Docker path in `nextjs-feature-based` remains correct.

## Coexistence with Superpowers

This skill activates during the **Cloudflare Workers deployment** phase.
Superpowers does not cover deployment — its scope ends at implementation and merge.
This skill is independent and has no overlap with any Superpowers skill.

If Superpowers is not installed, this skill works identically.
