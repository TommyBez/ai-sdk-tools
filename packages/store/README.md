# @ai-sdk-tools/store

High-performance chat state for the Vercel AI SDK. Drop in as a replacement for `@ai-sdk/react` and get O(1) lookups, message virtualization, transient data-part storage, and broadcast-safe hooks for any component.

---

## Highlights

- **3–5× fewer re-renders** thanks to batched updates and memoized selectors.
- **O(1) message lookups** via an internal id → index map.
- **Message virtualization** (`useVirtualMessages`) for massive histories.
- **Data-part awareness** – artifacts, agent status updates, and custom `data-*` parts are stored in a transient map so you can read them anywhere via `useDataPart`/`useDataParts`.
- **Same API as `@ai-sdk/react`** – `useChat` options (transport, experimental features) work unchanged.
- **Type-safe** – bring your own `UIMessage` generics.

---

## Installation

```bash
npm install @ai-sdk-tools/store
```

---

## Quick start (Next.js example)

```tsx
'use client';
import { Provider, useChat } from '@ai-sdk-tools/store';
import { DefaultChatTransport } from 'ai';

function Chat() {
  const { messages, input, handleInputChange, handleSubmit, status } = useChat({
    transport: new DefaultChatTransport({ api: '/api/chat' }),
  });

  return (
    <form className="chat" onSubmit={handleSubmit}>
      <section>
        {messages.map((message) => (
          <article key={message.id}>{message.content}</article>
        ))}
      </section>
      <footer>
        <input value={input} onChange={handleInputChange} />
        <button disabled={status === 'streaming'}>Send</button>
      </footer>
    </form>
  );
}

export default function Page() {
  return (
    <Provider initialMessages={[]}>
      <Chat />
    </Provider>
  );
}
```

`Provider` injects the chat store with optional `initialMessages` for SSR hydration. You can render multiple providers if you want independent chat instances.

---

## Why this store?

| Built-in optimization | Description |
| --- | --- |
| **Batching + scheduler** | Updates are batched on the main thread using `scheduler.postTask` / `requestAnimationFrame` to avoid thrashing during fast streams. |
| **Message index** | Lookups like `useMessageById` or `replaceMessageById` are constant time. |
| **Throttled streaming** | During streaming, the store writes to a transient buffer to keep React renders smooth (~60fps).
| **Memoized selectors** | `useSelector` caches expensive computations (word counts, metrics, etc.) keyed by your dependencies. |
| **Transient data map** | Any AI SDK `data-*` message part is stored in `_transientDataParts` so components can react to agent status, artifacts, rate limits, etc. |

---

## Core hooks

```ts
const chat = useChat(options);          // drop-in replacement for @ai-sdk/react
const messages = useChatMessages();     // returns the React-friendly messages array
const status = useChatStatus();         // 'ready' | 'streaming' | 'error'
const error = useChatError();
const chatId = useChatId();
```

Because these hooks read from a central store, you can call them from any component (no prop drilling).

### Data parts / artifacts

```tsx
import { useDataPart, useDataParts } from '@ai-sdk-tools/store';

function AgentStatus() {
  const [status] = useDataPart<{ status: string; agent: string }>('agent-status');
  return status ? <p>{status.agent}: {status.status}</p> : null;
}

function ArtifactOverview() {
  const { byType } = useDataParts();
  return Object.entries(byType).map(([type, artifacts]) => (
    <section key={type}>{type}: {artifacts.length}</section>
  ));
}
```

Any `data-*` part appended to an AI SDK message (artifacts, agent status, rate limits, custom data) is available through these hooks.

### Selectors & virtualization

```tsx
const message = useMessageById(id);
const count = useMessageCount();
const ids = useMessageIds();
const slice = useVirtualMessages(start, end); // renders just a window of messages

const unanswered = useSelector(
  'unanswered',
  (messages) => messages.filter((m) => m.role === 'user' && !m.metadata?.answered).length,
  [messages.length],
);
```

`useVirtualMessages` is perfect for chat panes with thousands of messages. `useSelector` memoizes results based on your dependency array.

### Actions

```ts
const actions = useChatActions();
actions.setMessages(newMessages);
actions.pushMessage(message);
actions.replaceMessageById(id, updatedMessage);
actions.reset();
```

Need the entire vanilla store? `useChatStore()` gives you the Zustand store, and `useChatStoreApi()` returns the store API (getState/setState) for advanced integrations.

---

## Custom stores

You can spin up additional chat stores (e.g., for multi-panel experiences) using the factory helpers.

```ts
import { createChatStore, createChatStoreCreator } from '@ai-sdk-tools/store';

const createStore = createChatStoreCreator();
export const customStore = createStore([]);

// In a component
const messages = useChatStore(customStore, (state) => state.messages);
```

Or create a full store instance manually via `createChatStore(initialMessages)` and pass it to components without the React context provider.

---

## TypeScript generics

Everything accepts generic `UIMessage` shapes, including metadata, data parts, and tool definitions.

```ts
interface WeatherMessage extends UIMessage<{ location: string }, { forecast?: ForecastPart }> {}

const { messages } = useChat<WeatherMessage>({
  transport: new DefaultChatTransport({ api: '/api/chat' }),
});
```

The entire store, hooks, and actions will be typed accordingly.

---

## Debugging

`configureDebug(Boolean)` or the `DEBUG` env variable toggle internal logging. Every production build keeps debug disabled unless you opt in.

```ts
import { configureDebug } from '@ai-sdk-tools/store';
configureDebug(process.env.NODE_ENV === 'development');
```

---

## API snapshot

| Export | Description |
| --- | --- |
| `Provider` | React provider wrapping the default chat store |
| `useChat`, `UseChatOptions`, `UseChatHelpers` | Enhanced replacement for `@ai-sdk/react`'s hook |
| `useChatMessages`, `useChatStatus`, `useChatError`, `useChatId` | Read core chat state |
| `useChatActions`, `useChatStore`, `useChatStoreApi`, `ChatStoreContext` | Access vanilla Zustand store + actions |
| `useMessageById`, `useMessageCount`, `useMessageIds`, `useVirtualMessages` | Message selectors |
| `useSelector` | Memoized selector helper |
| `useDataPart`, `useDataParts` | Work with AI SDK data parts |
| `createChatStore`, `createChatStoreCreator` | Build custom stores |
| `configureDebug`, `DebugLogger` | Optional logging |

See the source (`src/hooks.ts`, `src/use-chat.ts`, `src/use-data-parts.ts`) for complete signatures.

---

## Tips

- **SSR hydration** – pass `initialMessages` to `Provider` so the store lines up with server-rendered content.
- **Virtualization** – for long chats, render only the window that’s visible via `useVirtualMessages` and a virtualizer like `react-virtual`.
- **Derived data** – wrap heavy computations (word counts, analytics) with `useSelector` so they only recompute when dependencies change.
- **Artifacts + devtools** – because data parts live in the store, the devtools package can read them automatically.

---

## License

MIT © [Midday](https://midday.ai)
