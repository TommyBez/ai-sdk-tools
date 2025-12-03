# ai-sdk-tools

Everything from the AI SDK Tools ecosystem behind a single import. Install one dependency and gain access to agents, artifacts, cache, devtools, memory providers, and the high-performance store — all versioned together and fully typed.

```bash
npm install ai-sdk-tools
# plus the AI SDK + React stack you already use
npm install ai @ai-sdk/react react react-dom zod zustand
```

> Prefer a lighter bundle? You can always install `@ai-sdk-tools/<package>` directly. The umbrella package re-exports those modules without adding opinionated glue code.

---

## Included modules

| Module | Summary |
| --- | --- |
| `agents` | Multi-agent orchestration with routing, handoffs, memory, and telemetry |
| `artifacts` | Define & stream typed UI artifacts from tools into React |
| `cache` | Wrap any AI SDK tool (including streaming) with contextual caching |
| `devtools` | Real-time inspector for SSE streams, tool calls, agent graphs, and store state |
| `memory` | Provider-agnostic working memory + conversation history helpers |
| `store` | Drop-in replacement for `@ai-sdk/react` with heavily optimized Zustand stores |

All exports from the individual packages are available at the root:

```ts
import {
  Agent,
  artifact,
  cached,
  useChat,
  useArtifact,
  AIDevtools,
  InMemoryProvider,
} from 'ai-sdk-tools';
```

Tree-shaking works because the package simply re-exports ESM entry points.

---

## Quick start

```tsx
// app/api/chat/route.ts
import {
  Agent,
  handoff,
  cached,
  InMemoryProvider,
} from 'ai-sdk-tools';
import { tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const weather = cached(
  tool({
    description: 'Fetch weather for a city',
    parameters: z.object({ city: z.string() }),
    async execute({ city }) {
      return fetchJSON(`https://weather.api/${city}`);
    },
  }),
  { ttl: 15 * 60_000 },
);

const specialist = new Agent({
  name: 'Weather Specialist',
  model: openai('gpt-4o'),
  instructions: 'Answer weather questions with metric + imperial units.',
  tools: { weather },
});

const concierge = new Agent({
  name: 'Concierge',
  model: openai('gpt-4o-mini'),
  instructions: 'Triage all travel questions and route when needed.',
  handoffs: [specialist],
  memory: {
    provider: new InMemoryProvider(),
    workingMemory: { enabled: true, scope: 'user' },
  },
});

export async function POST(req: Request) {
  const payload = await req.json();
  return concierge.toUIMessageStream({
    message: payload.message,
    context: { chatId: payload.chatId, userId: payload.userId },
  });
}
```

```tsx
// app/chat/page.tsx
'use client';
import { useChat, useArtifact, AIDevtools, artifact } from 'ai-sdk-tools';
import { z } from 'zod';

const Forecast = artifact(
  'forecast',
  z.object({
    city: z.string(),
    hourly: z.array(z.object({ hour: z.string(), tempC: z.number() })),
  }),
);

export default function Chat() {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
  } = useChat({ api: '/api/chat' });
  const [{ data, status }] = useArtifact(Forecast);

  return (
    <>
      <form onSubmit={handleSubmit}>
        <ul>{messages.map((m) => <li key={m.id}>{m.content}</li>)}</ul>
        <input value={input} onChange={handleInputChange} />
      </form>

      {data && (
        <aside>
          <h2>{data.city}</h2>
          <p>Status: {status}</p>
          <ul>
            {data.hourly.map((row) => (
              <li key={row.hour}>{row.hour}: {row.tempC}°C</li>
            ))}
          </ul>
        </aside>
      )}

      {process.env.NODE_ENV === 'development' && <AIDevtools />}
    </>
  );
}
```

---

## Module cheat sheet

### Agents
- `Agent`, `handoff`, guardrail helpers, streaming writers, etc.
- Works with `toUIMessageStream` or plain `generate`.
- Add long-term context via `InMemoryProvider`, `RedisProvider`, `UpstashProvider`, or `DrizzleProvider` from the memory module.

### Artifacts
- `artifact`, `StreamingArtifact`, `useArtifact`, `useArtifacts`.
- Artifacts are versioned payloads emitted through AI SDK data parts and hydrated in React via the store.
- Pair with the store module (already included) to subscribe to artifact updates from any component.

### Cache
- `cached(tool, options?)`, `createCached(options)`, `cacheTools`.
- Handles plain tools **and** streaming tools (replays artifact parts + final text).
- Bring your own storage by passing `store` (must implement the `CacheStore` interface).

### Devtools
- `AIDevtools`, `useAIDevtools`, `StreamInterceptor`, parser utilities, and rich type definitions.
- Captures SSE streams automatically, groups tool calls, throttles noisy events, and inspects `@ai-sdk-tools/store`.

### Memory
- Type definitions + utility helpers (`formatWorkingMemory`, `formatHistory`, `DEFAULT_TEMPLATE`).
- Providers live under subpath exports:  
  `import { InMemoryProvider } from 'ai-sdk-tools/memory/in-memory';`  
  `import { RedisProvider } from 'ai-sdk-tools/memory/redis';`  
  `import { UpstashProvider } from 'ai-sdk-tools/memory/upstash';`  
  `import { DrizzleProvider } from 'ai-sdk-tools/memory/drizzle';`

### Store
- `useChat`, `Provider`, selectors (`useMessageById`, `useDataPart`, `useVirtualMessages`, …) and the optimized Zustand store factory.
- Compatible with `@ai-sdk/react` transports; just swap the hook import.

---

## Server + client split

| Where | Typical imports |
| --- | --- |
| **Server / Route handlers** | `Agent`, `artifact`, `cached`, memory providers |
| **Client Components** | `useChat`, `useArtifact`, `useDataPart`, `AIDevtools` |

Because everything comes from one package you can still tree-shake (ESM re-exports). Make sure your bundler is configured for ESM (Next.js, Vite, Bun all work out of the box).

---

## Using individual packages

Need only a subset? Install the specific package:

```bash
npm install @ai-sdk-tools/agents
npm install @ai-sdk-tools/store
```

The APIs are identical — the umbrella package does not wrap or fork anything. Mixing and matching (e.g., `Agent` from `@ai-sdk-tools/agents` with `useChat` from `ai-sdk-tools`) is fully supported.

---

## Requirements

- Node.js 18+
- Vercel AI SDK v5 or newer
- React 18+ for UI hooks/devtools

## Useful links

- [Examples](https://github.com/midday-ai/ai-sdk-tools/tree/main/apps/example)
- [Issues](https://github.com/midday-ai/ai-sdk-tools/issues)
- [Discussions](https://github.com/midday-ai/ai-sdk-tools/discussions)

## License

MIT © [Midday](https://midday.ai)

