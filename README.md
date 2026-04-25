# Speed Search

A personal experiment to test the speed of a prefix-based search algorithm using Redis sorted sets as the backing store.

The core idea: instead of a traditional full-text search or LIKE query, country names are pre-indexed as all their possible prefixes in an Upstash Redis sorted set. At query time, a single `ZRANK` + `ZRANGE` call retrieves matching completions — demonstrating sub-millisecond read latency even at the edge.

## How the algorithm works

During seeding (`src/lib/seed.ts`), each country name is uppercased and broken into every prefix:

```
"FRANCE" → ["F", "FR", "FRA", "FRAN", "FRANC", "FRANCE*"]
```

All prefixes are stored as members of a Redis sorted set (`terms`) with score `0`. The trailing `*` marks a complete word. Because members in a sorted set are ordered lexicographically when scores are equal, all prefixes of a term cluster together.

At search time (`/api/search?q=`):

1. `ZRANK` finds the position of the query string in the sorted set.
2. `ZRANGE` fetches the next 50 members from that position.
3. Members are collected while they still start with the query; the loop stops at the first non-matching entry.
4. Only members ending with `*` (complete words) are returned, with the `*` stripped.

The response includes both the matched country names and the time taken in milliseconds.

## Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14, Tailwind CSS, shadcn/ui (`cmdk`) |
| API | Hono on Cloudflare Workers (edge runtime) |
| Database | Upstash Redis (serverless Redis over HTTP) |
| Deployment | Wrangler (Cloudflare Workers) |

## Project structure

```
src/
  app/
    api/[[...route]]/route.ts   # Hono search endpoint
    page.tsx                    # Search UI
  lib/
    seed.ts                     # One-time Redis population script
```

## Running locally

```bash
pnpm install
pnpm dev
```

## Seeding the database

Run once to populate Upstash Redis with the country prefix index:

```bash
npx tsx src/lib/seed.ts
```

## Deploying the API

The API is deployed separately to Cloudflare Workers via Wrangler:

```bash
pnpm deploy
```

Set `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN` in `wrangler.toml` (or as Cloudflare Worker secrets for production).
