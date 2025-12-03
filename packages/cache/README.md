# @ai-sdk-tools/cache

[![npm version](https://badge.fury.io/js/@ai-sdk-tools%2Fcache.svg)](https://badge.fury.io/js/@ai-sdk-tools%2Fcache)

Drop-in caching for AI SDK tools (regular or streaming). Wrap any tool with `cached()` and the result is stored in-memory (LRU) by default, or in any backend that implements a tiny `CacheStore` interface.

---

## Installation

```bash
npm install @ai-sdk-tools/cache
```

### Peer dependency

You already have the AI SDK installed, but for completeness:

```bash
npm install ai
```

---

## Why cache tools?

- Avoid repeated API calls (weather lookups, retrieval queries, pricing data).
- Accelerate multi-step workflows where the same tool is invoked multiple times in a single run.
- Keep streaming experiences smooth — cached streams replay the exact artifacts/data parts that were produced the first time.
- Share results across users or scopes by incorporating context into the cache key.

---

## Quick start

```ts
import { cached } from '@ai-sdk-tools/cache';
import { tool } from 'ai';
import { z } from 'zod';

const weather = tool({
  description: 'Fetch current weather',
  parameters: z.object({ city: z.string() }),
  async execute({ city }) {
    return fetchJSON(`https://weather.api/${city}`);
  },
});

const cachedWeather = cached(weather, {
  ttl: 10 * 60_000,         // 10 minutes
  cacheKey: () => 'global', // optional context string
});

// Use inside generateText / streamText like any other tool
await generateText({
  model: openai('gpt-4o'),
  tools: { weather: cachedWeather },
  messages: [{ role: 'user', content: 'Weather in NYC?' }],
});
```

### Share configuration with `createCached`

```ts
import { createCached } from '@ai-sdk-tools/cache';

const cached = createCached({
  ttl: 5 * 60_000,
  cacheKey: () => {
    const session = getSession();
    return `team:${session.teamId}`;
  },
  onHit(key) {
    metrics.increment('tool.cache.hit', { key });
  },
});

export const tools = {
  docsSearch: cached(docsSearchTool),
  summarize: cached(summarizeTool),
};
```

### Streaming tools & artifacts

`cached()` automatically records every streaming yield (final chunk + data parts) and replays them when a cache hit occurs.

```ts
const streamReport = cached(
  tool({
    description: 'Build a chart',
    parameters: z.object({ id: z.string() }),
    async *execute(params, executionOptions) {
      const writer = getWriter(executionOptions);
      const artifact = ReportArtifact.stream({ stage: 'loading' }, writer);

      await artifact.update({ stage: 'processing' });
      yield { text: 'Working…' };

      await artifact.complete({
        stage: 'complete',
        sections: buildSections(params.id),
      });
    },
  }),
);
```

First call captures every artifact update and text chunk. Subsequent calls skip the expensive work and replay the exact same events to the client instantly.

---

## Configuration

`CacheOptions` (usable in both `cached()` and `createCached()`):

| Option | Description |
| --- | --- |
| `ttl` | Time-to-live in milliseconds (default `5 minutes` for LRU) |
| `maxSize` | Maximum entries for the default LRU store (default `1000`) |
| `store` | Custom store implementing the `CacheStore` interface |
| `keyGenerator` | `(params, context?) => string` – override serialization |
| `cacheKey` | `() => string` – supply contextual key fragments (user/team) |
| `shouldCache` | `(params, result) => boolean` – skip caching for some responses |
| `onHit` / `onMiss` | Callbacks for analytics/logging |
| `debug` | Verbose logging to stdout |

The default store is an in-memory LRU map. Pass your own `store` for Redis, DynamoDB, etc.

---

## Implementing a custom store

Any object that satisfies `CacheStore` works:

```ts
import type { CacheStore, CacheEntry } from '@ai-sdk-tools/cache';
import { Redis } from '@upstash/redis';

const redis = Redis.fromEnv();

const redisStore: CacheStore = {
  async get(key) {
    const raw = await redis.get<string>(key);
    return raw ? JSON.parse(raw) as CacheEntry : undefined;
  },
  async set(key, entry) {
    await redis.set(key, JSON.stringify(entry));
  },
  async delete(key) {
    await redis.del(key);
    return true;
  },
  async clear() {
    // up to you: iterate keys or flush a prefix
  },
  async has(key) {
    return Boolean(await redis.exists(key));
  },
  async size() {
    // approximate implementation
    return 0;
  },
  async keys() {
    return [];
  },
};

const cached = createCached({ store: redisStore, ttl: 15 * 60_000 });
```

`CacheEntry` is a simple `{ result, timestamp, key }` payload — serialize however you like.

---

## Multi-tenant cache keys

```ts
const getCacheContext = () => {
  const session = auth();
  return `team:${session.teamId}:user:${session.userId}`;
};

const recommend = cached(recommendationsTool, {
  cacheKey: getCacheContext,
  ttl: 30 * 60_000,
});
```

`cacheKey()` runs for each call; combine it with `keyGenerator` if you need fine-grained control over serialization.

---

## Working with multiple tools

```ts
import { cacheTools } from '@ai-sdk-tools/cache';

const { searchDocs, summarize, translate } = cacheTools(
  { searchDocs: docsTool, summarize: summarizer, translate: translationTool },
  {
    ttl: 5 * 60_000,
    cacheKey: () => `workspace:${currentWorkspace()}`,
  },
);
```

`cacheTools` applies the same configuration to each tool and returns an object with cached versions.

---

## Stats & maintenance

Every cached tool gains helper methods:

```ts
const cachedTool = cached(expensiveTool);

cachedTool.getStats();
// { hits: 4, misses: 1, hitRate: 0.8, size: 3, maxSize: 1000 }

cachedTool.clearCache();           // clear everything
cachedTool.clearCache('a-key');    // clear just that entry
await cachedTool.isCached(params); // boolean (async for async stores)
cachedTool.getCacheKey(params);    // introspect the derived key
```

Use these to build admin panels or to invalidate cached data when upstream systems change.

---

## Debugging

Enable `debug: true` to log cache behavior:

```ts
const cached = createCached({
  debug: process.env.NODE_ENV === 'development',
  onHit: (key) => console.log('[cache hit]', key),
  onMiss: (key) => console.log('[cache miss]', key),
});
```

Streaming caches also log how many chunks/artifacts were replayed so you can verify behavior locally.

---

## API summary

### `cached(tool, options?)`
Wrap a single tool immediately. Returns a cached tool instance (with stats + helpers).

### `createCached(options?)`
Returns a function you can reuse to wrap many tools with the same defaults.

### `cacheTools(record, options?)`
Wrap multiple tools at once and get back an object with cached versions.

### Types
- `CacheOptions`, `CacheStore`, `CacheEntry`, `CacheStats`, `CachedTool<T extends Tool>`.

---

## License

MIT © [Midday](https://midday.ai)