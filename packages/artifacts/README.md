# @ai-sdk-tools/artifacts

Stream structured, type-safe payloads from AI tools into React without prop drilling. Define an artifact once, update it incrementally on the server, and consume it anywhere in your UI through hooks powered by `@ai-sdk-tools/store`.

---

## Installation

```bash
npm install @ai-sdk-tools/artifacts @ai-sdk-tools/store
# also install the AI SDK + React stack you already use
npm install ai @ai-sdk/react react react-dom zod
```

Why the store? The store package keeps a single copy of the AI SDK message stream, deduplicates renders, and exposes selectors that let `useArtifact` / `useArtifacts` read the latest data parts without prop drilling.

---

## What is an artifact?

- A named, typed payload (backed by a Zod schema) that travels through AI SDK `data-*` parts.
- Includes metadata: `status`, `progress`, `version`, timestamps, and optional error text.
- Can be updated incrementally (`update`), marked complete (`complete`), cancelled/errored, or even given a `timeout`.
- Every update is stored in the chat store so you can browse versions or replay history.

---

## End-to-end example

### 1. Define the artifact schema

```ts
import { artifact } from '@ai-sdk-tools/artifacts';
import { z } from 'zod';

export const BurnRateArtifact = artifact(
  'burn-rate',
  z.object({
    company: z.string(),
    stage: z.enum(['collecting', 'processing', 'complete']).default('collecting'),
    monthlyBurn: z.number().nullable(),
    runwayMonths: z.number().nullable(),
    series: z.array(z.object({ month: z.string(), value: z.number() })).default([]),
  }),
);
```

### 2. Stream from a tool / route handler

```ts
import { tool } from 'ai';
import { getWriter } from '@ai-sdk-tools/artifacts';

export const analyzeBurnRate = tool({
  description: 'Analyze company burn rate',
  parameters: z.object({ companyId: z.string() }),
  async *execute(params, executionOptions) {
    const writer = getWriter(executionOptions);
    const artifact = BurnRateArtifact.stream(
      {
        company: params.companyId,
        stage: 'collecting',
        monthlyBurn: null,
        runwayMonths: null,
      },
      writer,
    );

    const transactions = await fetchTransactions(params.companyId);
    yield { text: 'Crunching ledger…' };

    await artifact.update({
      stage: 'processing',
      series: aggregate(transactions),
      monthlyBurn: 84200,
      runwayMonths: 13,
    });

    await artifact.complete();
    yield { text: 'Report ready', forceStop: true };
  },
});
```

`getWriter` extracts the `UIMessageStreamWriter` from `executionOptions.experimental_context`. If a writer is missing the helper throws, making it clear that the tool must run inside a streaming request.

### 3. Route handler (Next.js example)

```ts
import { createUIMessageStream, createUIMessageStreamResponse } from 'ai';
import { streamText } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const stream = createUIMessageStream({
    execute: ({ writer }) => {
      const result = streamText({
        model: openai('gpt-4o'),
        messages,
        tools: { analyzeBurnRate },
      });

      writer.merge(result.toUIMessageStream());
    },
  });

  return createUIMessageStreamResponse({ stream });
}
```

### 4. React consumption (any component)

```tsx
'use client';
import { useArtifact, useArtifacts } from '@ai-sdk-tools/artifacts/client';
import { useChat } from '@ai-sdk-tools/store';

export function Chat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    api: '/api/chat',
  });
  const [{ data, status, progress }] = useArtifact(BurnRateArtifact, {
    onComplete: (payload) => toast.success(`${payload.company} ready`),
  });

  return (
    <>
      <ul>{messages.map((m) => <li key={m.id}>{m.content}</li>)}</ul>
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
      </form>
      {data && (
        <aside>
          <p>{data.company} — {status}</p>
          {progress !== undefined && <p>{Math.round(progress * 100)}%</p>}
        </aside>
      )}
    </>
  );
}

export function ArtifactCanvas() {
  const [{ current, types, activeType }, { setValue }] = useArtifacts({
    onData(type, latest) {
      analytics.track('artifact_updated', { type, status: latest.status });
    },
  });

  if (!current) return <EmptyState types={types} />;
  return (
    <>
      <Tabs tabs={types} active={activeType} onSelect={setValue} />
      <Renderer artifact={current} />
    </>
  );
}
```

`useArtifact` and `useArtifacts` both return `[state, actions]`. Actions let you delete a specific artifact (`actions.delete(id)`), dismiss or restore a type, or control which artifact is considered “active” (useful for dashboards).

---

## Key concepts

### Writer & data parts
- Artifacts travel through `UIMessageStreamWriter.write({ type: 'data-artifact-<id>', ... })`.
- `getWriter(executionOptions)` reads the writer from the AI SDK execution context so your tool code stays clean.
- Every artifact share uses the same store as chat messages, meaning you can hydrate the UI anywhere (no prop drilling or React context gymnastics).

### Streaming lifecycle
- `artifact(id, schema)` returns helpers:
  - `create(initial?)` – validate data immediately without streaming.
  - `stream(initial, writer)` – returns a `StreamingArtifact`.
  - `validate`, `isValid` – runtime checks when data comes from an arbitrary source.
- `StreamingArtifact` exposes:
  - `update(partial)`, `complete(final?)`, `error(message)`, `cancel()`.
  - `progress` getter/setter (updating it automatically emits a new version).
  - `timeout(ms)` to auto-fail if no completion occurs.

### Hooks
- `useArtifact(definition, options?)`
  - `options.version` lets you inspect historical versions while still receiving callbacks for the latest data.
  - Returns `{ data, status, progress, error, isActive, hasData, versions }` plus actions `{ delete }`.
- `useArtifacts(options?)`
  - Track all artifacts grouped by type, react to new data via `onData`, and control UI state using `setValue`, `dismiss`, `restore`.
  - Supports `include`/`exclude`, controlled `value`, and externally managed dismissed lists.

### Status & versioning

| Status | Meaning |
| --- | --- |
| `idle` | Artifact exists but has no payload yet |
| `loading` | Initial payload en route |
| `streaming` | Receiving incremental updates |
| `complete` | Final payload yielded |
| `error` | `error(message)` or `cancel()` was called |

Every update increments `version` and stamps `updatedAt`. `useArtifact` keeps a full history so you can build rewind/compare features.

---

## API reference

### `artifact(id: string, schema: ZodSchema<T>)`
Returns an object with:

| Method | Description |
| --- | --- |
| `create(initial?: Partial<T>)` | Produce an `ArtifactData<T>` immediately |
| `stream(initial: Partial<T>, writer)` | Create a `StreamingArtifact<T>` |
| `validate(data: unknown)` | Throws if data fails schema |
| `isValid(data: unknown)` | Type guard |

### `StreamingArtifact<T>`

```ts
const instance = BurnRateArtifact.stream(initial, writer);
await instance.update(partial);
await instance.complete(finalData?);
await instance.error('Something went wrong');
instance.progress = 0.6;
instance.timeout(10_000);
```

### `getWriter(executionOptions?: { experimental_context?: any })`
Extracts the `UIMessageStreamWriter` from the AI SDK execution options. Throws if missing.

### Hooks

```ts
const [state, actions] = useArtifact(BurnRateArtifact, {
  onUpdate(data) {},
  onComplete(data) {},
  onError(message, data) {},
  onProgress(progress, data) {},
  onStatusChange(status, prevStatus) {},
  version: 0, // optional
});

const [collection, actions] = useArtifacts({
  include: ['burn-rate', 'pipeline-report'],
  onData(type, artifact) {},
  value: activePanel,      // controlled mode
  onChange: setActivePanel,
  dismissed, onDismissedChange,
});
```

- `UseArtifactReturn<T>` = `{ data, status, progress, error, isActive, hasData, versions, currentIndex }`.
- `UseArtifactActions` = `{ delete(artifactId) }`.
- `UseArtifactsReturn` = metadata for all types plus `latestByType`, `current`, `activeType`, etc.
- `UseArtifactsActions` = `{ setValue, dismiss, restore }`.

### Types & errors
- `ArtifactData`, `ArtifactStatus`, `ArtifactConfig`, `ArtifactCallbacks`.
- `ArtifactError` – custom error with `code` for upstream handling.

---

## Tips & advanced usage

- **Notifications & analytics** – subscribe to `useArtifacts({ onData })` in a top-level component to log or toast when artifacts change, even if the rendering happens elsewhere.
- **Multiple stores** – all hooks accept `storeId`. Instantiate additional chat stores via `@ai-sdk-tools/store` if you need isolated timelines (e.g., multiple assistants on the same page).
- **Deleting artifacts** – `actions.delete(artifactId)` removes every matching data part from the store (useful when a user closes a panel).
- **Controlled canvases** – pass `value` and `onChange` to `useArtifacts` to drive tabs, carousels, or multi-canvas dashboards while still receiving automatic auto-open behavior for new types.
- **Server-only validation** – use `artifact.validate` before persisting or merging artifacts coming from untrusted sources.

---

## Examples

Browse the ready-made examples under `packages/artifacts/src/examples/`:

- `burn-rate-example.ts` – streaming analysis with progress updates
- `usage-example.tsx` – React consumption with `useArtifact`
- `use-artifacts-example.tsx` – multi-type canvas using `useArtifacts`
- `typed-context-example.ts` – combining artifacts with typed execution context

---

## License

MIT © [Midday](https://midday.ai)
