![AI SDK Devtools](image.png)

<br />

# AI SDK Devtools

Live diagnostics for AI SDK streams. See every SSE event, inspect tool calls, visualize agent handoffs, and (optionally) peek inside `@ai-sdk-tools/store` state.

---

## Installation

```bash
npm install @ai-sdk-tools/devtools
# optional (state tab)
npm install @ai-sdk-tools/store
```

The store package enables the “State” tab, but the devtools panel works fine without it.

---

## Quick start

```tsx
'use client';
import { AIDevtools } from '@ai-sdk-tools/devtools';

export function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <>
      {children}
      {process.env.NODE_ENV === 'development' && <AIDevtools />}
    </>
  );
}
```

That’s it. Whenever a page uses `useChat`, `useChatMessages`, or streams UI events from the AI SDK, the devtools panel listens and renders them in real time.

---

## What you get

- **Stream inspector** – watch `message-start`, `text-delta`, `tool-call-*`, `finish` events as they happen.
- **Tool call details** – duration, args, result payloads, and errors.
- **Agent flow** – specialized panes for `@ai-sdk-tools/agents` data parts (`agent-status`, `agent-handoff`, rate limits).
- **Performance metrics** – characters/sec, token deltas, latency between tool steps.
- **Filtering/search** – filter by event type, tool name, or free-text search.
- **State tab (optional)** – if `@ai-sdk-tools/store` is present, inspect chat messages, artifacts, and transient data parts.

The panel is resizable and can be docked to the bottom, right, or overlay.

---

## Configuration

```tsx
<AIDevtools
  enabled={process.env.NODE_ENV === 'development'}
  maxEvents={750}
  config={{
    position: 'bottom',          // 'bottom' | 'right' | 'overlay'
    height: 360,
    streamCapture: {
      enabled: true,             // patch fetch() to capture SSE
      endpoints: ['/api/chat'],  // list of endpoints to intercept
      autoConnect: true,
    },
    throttle: {
      enabled: true,
      interval: 100,             // ms between batched UI updates
      includeTypes: ['text-delta'],
    },
  }}
  debug={false}                  // verbose console logging
/>
```

- `maxEvents` – clips history to avoid slow UIs.
- `streamCapture` – uses the built-in `StreamInterceptor` to patch `fetch()` and listen to SSE streams without any extra code.
- `throttle` – tame noisy events like text deltas.

---

## `useAIDevtools` hook

Want to build your own dashboard or show summaries elsewhere in the UI? Use the hook directly.

```tsx
import { useAIDevtools } from '@ai-sdk-tools/devtools';

function Stats() {
  const {
    events,
    clearEvents,
    toggleCapturing,
    filterEvents,
    getEventStats,
    getUniqueToolNames,
  } = useAIDevtools({
    maxEvents: 500,
    onEvent: (event) => analytics.track('ai_event', event),
  });

  const errors = filterEvents(['error']);
  const stats = getEventStats();

  return (
    <section>
      <p>Total events: {events.length}</p>
      <p>Errors: {errors.length}</p>
      <p>Tools seen: {getUniqueToolNames().join(', ')}</p>
      <button onClick={toggleCapturing}>Pause</button>
      <button onClick={clearEvents}>Clear</button>
    </section>
  );
}
```

The hook powers the panel, so anything the panel can do you can do in your own components.

---

## Stream interceptor (standalone)

If you want to control when interception happens (e.g., only around specific fetch calls), use `StreamInterceptor` directly:

```ts
import { StreamInterceptor } from '@ai-sdk-tools/devtools';

const interceptor = new StreamInterceptor({
  endpoints: ['/api/chat'],
  enabled: true,
  onEvent(event) {
    myStore.add(event);
  },
});

interceptor.patch();    // wraps window.fetch
// ...
interceptor.unpatch();
```

---

## Devtools config reference

| Prop | Description |
| --- | --- |
| `enabled` | Show/hide the panel |
| `maxEvents` | How many events to keep before trimming |
| `modelId` | Optional model label (used in UI badges) |
| `config.position` | `'bottom' | 'right' | 'overlay'` |
| `config.height`, `config.width` | Initial size (px) |
| `config.streamCapture` | `{ enabled, endpoints[], autoConnect }` |
| `config.throttle` | `{ enabled, interval, includeTypes?, excludeTypes? }` |
| `debug` | Logs internal devtool actions to `console` |

---

## Event types

Captured out of the box:

- `message-start`, `message-chunk`, `message-complete`
- `text-start`, `text-delta`, `text-end`
- `tool-call-start`, `tool-call-result`, `tool-call-error`
- `error`, `finish`, `stream-done`
- Agent-specific: `agent-start`, `agent-step`, `agent-finish`, `agent-handoff`, `agent-complete`, `agent-error`

Anything emitted as an AI SDK data part (e.g., `data-artifact-*`, `data-agent-*`) also shows up under the “Data” panel.

---

## Requirements

- React 18+ (Client Component)
- Runs in any environment where `window.fetch` is available (Next.js, Vite, etc.)
- Optional `@ai-sdk-tools/store` for the state tab

---

## License

MIT © [Midday](https://midday.ai)