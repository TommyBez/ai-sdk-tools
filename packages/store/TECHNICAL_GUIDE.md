# Store package: deep technical guide

This guide documents the **internal architecture**, **performance model**, and **extension patterns** of `@ai-sdk-tools/store`.

It’s written for engineers who want to:

- understand *why* the package behaves the way it does (batching, throttling, SSR hydration behavior)
- integrate it into a complex React app (multiple chat instances, custom stores, derived state)
- extend the underlying Zustand store safely (middleware, additional cached selectors, derived indexes)

## What this package is (and is not)

`@ai-sdk-tools/store` is a **React + Zustand** integration layer around `@ai-sdk/react`’s `useChat`.

- It **wraps** `@ai-sdk/react`’s `useChat` (called `useOriginalChat` in the implementation).
- It **syncs** the chat helpers and state into a **Zustand vanilla store** stored in React Context.
- It exposes a set of **high-performance hooks** that subscribe to small slices of that store.

It is **not** a full replacement for the AI SDK runtime; it assumes your app still uses AI SDK types and transports. This package focuses on **state management and render performance**.

## Export surface (public API)

All exports are re-exported from `packages/store/src/index.ts`:

- **Provider / store factory**
  - `Provider`
  - `createChatStore`, `createChatStoreCreator`
  - `ChatStoreContext` (for advanced usage only)
  - `StoreState` (store state type)
  - `ChatActions` (convenience action shape)
- **Hooks**
  - `useChat` (enhanced wrapper around `@ai-sdk/react`)
  - `useChatStore`, `useChatStoreApi`
  - `useChatMessages`, `useChatStatus`, `useChatError`, `useChatId`
  - `useMessageById`, `useMessageIds`, `useMessageCount`, `useVirtualMessages`
  - `useChatActions`, `useChatReset`
  - `useSelector` (memoized derived selector with dependency array)
- **Streaming “data part” helpers**
  - `useDataParts`, `useDataPart` (+ types)
- **Debugging**
  - `debug`, `configureDebug`, `DebugLogger`
- **Types**
  - `UIMessage` (re-exported from `@ai-sdk/react`)

## High-level data flow

At runtime there are two interacting state machines:

1. `@ai-sdk/react` `useChat` internal state (networking, streaming, input, etc.)
2. The Zustand store state in this package (messages, status, error, id + performance caches)

The wrapper hook **bridges** these:

```text
@ai-sdk/react useChat()
  ├─ produces: { id, messages, status, error, sendMessage, ... }
  └─ calls onData(dataPart) during streaming
          │
          ▼
@ai-sdk-tools/store useChat()
  ├─ wraps onData to also record transient data parts
  ├─ syncs state into the Zustand store via _syncState()
  └─ returns the same helpers BUT uses store.messages as source of truth
          │
          ▼
Zustand vanilla store
  ├─ canonical state: { id, messages, status, error }
  └─ performance caches: throttled snapshot, id index, memoized selectors, transient data map
          │
          ▼
High-performance hooks (useChatMessages, useMessageById, useSelector, …)
```

## Provider and store lifecycle

### The `Provider` creates exactly one store instance per mount

`Provider` creates the store in a `ref` so it is stable for the lifetime of the component instance:

- On first render: it creates a store via `createChatStore(initialMessages)` **or** uses the supplied `store` prop.
- On subsequent renders: it reuses the same store instance.

This is critical: **Zustand subscriptions depend on stable store identity**. Re-creating the store would reset state and cause re-subscriptions.

### Resetting the store for a “new chat”

If you want a completely new store instance, remount the provider:

- change a `key` on `<Provider key={chatKey} ...>` (see `apps/example/src/app/custom-store/custom-store-provider.tsx`)

### “use client” implications (Next.js App Router)

The core implementation files (`hooks.ts`, `use-data-parts.ts`) start with `"use client"`. That means:

- `Provider` must be used inside a **client component** boundary.
- All hooks must be called from **client components**.

SSR is still relevant because you can render initial messages on the server and pass them down as `initialMessages`, then hydrate on the client without clobbering them (details below).

## Store state model: `StoreState`

`StoreState<TMessage extends UIMessage>` contains:

### Canonical chat state

- `id: string | undefined`
- `messages: TMessage[]`
- `status: ChatStatus` (from the `ai` package)
- `error: Error | undefined`

### Performance caches / internal fields

These fields exist purely to speed up selectors and reduce render pressure:

- `_throttledMessages: TMessage[] | null`
  - a throttled snapshot of `messages` used for most reads, updated at ~60fps
- `_messageIndex: MessageIndex<TMessage>`
  - a `Map`-backed index for O(1) `getById()` and `getIndexById()`
- `_memoizedSelectors: Map<string, { result: any; deps: any[] }>`
  - cache for `useSelector()` / `getMemoizedSelector()`
- `_transientDataParts: Map<string, any>`
  - transient, out-of-band streaming payloads (used by `useDataPart`)

### Mutating actions

Core setters:

- `setId`, `setMessages`, `setStatus`, `setError`
- `setNewChat(id, messages)` (reset state as a new chat)
- `pushMessage`, `popMessage`, `replaceMessage`, `replaceMessageById`
- `reset()` (clears everything, including caches and transient data)

### Optimized getters

These are methods on the store state that hide implementation details like throttling:

- `getThrottledMessages()`:
  - returns `_throttledMessages` if present, otherwise returns `messages`
- `getInternalMessages()`:
  - returns `messages` without throttling (rarely needed in UI)
- `getMessageById(id)` / `getMessageIndexById(id)`:
  - reads from `_messageIndex` (Map-backed)
- `getMessagesSlice(start, end?)`:
  - slices the throttled snapshot for virtualization
- `getMessageIds()` / `getLastMessageId()` / `getMessageCount()`
- `getMemoizedSelector(key, selector, deps)`:
  - caches derived computations keyed by `key` + dependency array

## Performance model: what makes it faster

The store is designed to reduce re-renders in “hot paths” like streaming assistant tokens.

### 1) Batched updates with priority scheduling

All mutating actions route through an internal `batchUpdates(callback, priority?)` queue.

- On the server (`window` undefined): the callback executes immediately.
- In the browser:
  - callbacks are pushed into a queue
  - execution is scheduled using (in order):
    - `scheduler.postTask` if available
    - `requestAnimationFrame`
    - `setTimeout(…, 0)`
  - queued callbacks are sorted by `priority` descending and executed in that order

Why this matters:

- Streaming can trigger dozens of message updates per second.
- Batching collapses multiple synchronous state changes into fewer React render passes.
- Priority `1` is used for “streaming smoothness” updates (see below).

### 2) Throttled message snapshots (60fps)

The store maintains `_throttledMessages` as the typical “read model” for UI.

- For non-streaming states, message updates are throttled to `MESSAGES_THROTTLE_MS = 16` (~60fps).
- For streaming state (`status === "streaming"`), the store bypasses the throttle and updates the snapshot immediately with higher priority.

This yields:

- smooth token streaming (no visible “chunking”)
- reduced selector churn when messages update extremely frequently

Important detail:

- `_messageIndex` is updated when `_throttledMessages` is updated.
- This means O(1) lookups track the read model.
  - In non-streaming mode they can be *up to ~16ms behind* `messages`, which is generally acceptable for UI reads.

### 3) O(1) message lookup with `MessageIndex`

`MessageIndex` maintains two maps:

- `idToMessage: Map<string, TMessage>`
- `idToIndex: Map<string, number>`

Updates rebuild the index from the (snapshot) message list.

Implications:

- `useMessageById(id)` can be constant-time instead of `messages.find(...)`.
- `replaceMessageById(id, message)` avoids a linear scan (it uses the cached index).

### 4) Memoized derived selectors (`useSelector`)

`useSelector(key, selector, deps)` is a convenience hook for expensive derived computations.

It delegates to `StoreState.getMemoizedSelector(key, selectorFn, deps)`.

How caching works:

- The cache is keyed by **string key**.
- Dependencies are compared with:
  - same length
  - `JSON.stringify(previousDeps) === JSON.stringify(nextDeps)` (fast enough for small dep arrays; be careful with large objects)
- The store clears `_memoizedSelectors` on most message mutations (set/push/pop/replace/sync).

Rules of thumb:

- Use small, JSON-stable dependency arrays (numbers, strings, booleans).
- Treat `key` as a namespace to avoid collisions (e.g. `"analytics:userMessageCount"`).

### 5) React hook-level optimizations

The hooks use:

- `useShallow` from Zustand to avoid re-renders when objects are shallow-equal
- stable selector functions (`statusSelector`, `errorSelector`, …) to avoid recreating functions
- `useCallback` selectors when parameters exist (e.g. `useMessageById(messageId)`)

### 6) Freeze detector (dev-time signal)

`hooks.ts` starts a “freeze detector” in the browser:

- every animation frame it checks if the main thread was blocked beyond a threshold (~80ms)
- when it detects a “freeze”, it logs a warning via `debug.warn(...)` including the last action label

This is intended to help diagnose performance regressions while streaming or rendering large histories.

## The enhanced `useChat` wrapper

`useChat` in this package wraps `@ai-sdk/react`’s `useChat` and synchronizes it into the store.

### Why a wrapper is needed

`@ai-sdk/react`’s `useChat` keeps state in the component where it’s called. That makes it hard to:

- read chat state in unrelated components without prop drilling
- optimize rendering for large histories and high-frequency streaming updates

This package instead makes the store the “shared truth” for messages and exposes fine-grained subscriptions.

### Provider requirement

Even if you pass a custom store into `useChat({ store })`, the implementation still calls `useChatStoreApi()` to obtain the context store for fallback.

So in practice:

- **you must mount `<Provider>`** above `useChat` calls

### State sync strategy

On every relevant change in the original chat helpers, an effect runs and syncs:

- `id`, `status`, `error`
- `messages` *conditionally* (see hydration behavior below)
- plus helper functions: `sendMessage`, `regenerate`, `stop`, `resumeStream`, `addToolResult`, `setMessages`, `clearError`

Sync mechanism:

- If the store is a vanilla Zustand store (`getState` exists), it calls `store.getState()._syncState(partial)`.
- Otherwise it tries `store._syncState(partial)`.
- Otherwise it falls back to `store.setState(partial)`.

### Batching the sync (`enableBatching`)

The wrapper supports `enableBatching` (default `true`):

- when enabled in the browser, it schedules the sync in `requestAnimationFrame`
- when disabled, it calls sync immediately

This reduces “sync thrash” when multiple parts of the chat helpers change rapidly.

### SSR hydration: preserving server-rendered messages

There is a specific guard:

- if the current store already has messages
- and the wrapped `useChat` has *zero* messages
- then the wrapper will **not** sync `messages` (it syncs id/status/error/functions only)

This prevents a common Next.js hydration issue:

- server renders `initialMessages`
- client mounts `useChat` which starts with `messages=[]`
- without the guard, the client would overwrite the server messages and cause flicker/loss

### Messages returned from `useChat`

This package’s `useChat` returns:

- all the original `useChat` helpers
- but `messages` are sourced from the store subscription (`useStore(store, state => state.messages)`)

This ensures:

- components using `useChat` re-render based on store updates (including throttling/batching behavior)
- the store remains the message source of truth

## Action helpers and safe defaults

`useChatActions()` returns a stable object with:

- state mutation actions
- plus chat helper functions (sendMessage, stop, …)

When chat helpers aren’t configured yet (i.e. `useChat` hasn’t synced them), it uses fallback functions that **warn** via `debug.warn(...)` rather than throwing.

This prevents accidental infinite loops or crashes when calling actions before `useChat` runs.

## Data parts: `useDataParts` and `useDataPart`

AI SDK messages can contain structured “parts”, and some parts represent **data payloads** rather than user-visible text.

This package supports two ways to access those:

### Message-embedded data parts

`useDataParts()` scans `messages` and extracts parts where:

- `part.type` starts with `"data-"`
- and `part.data` exists

It returns:

- `byType`: a map keyed by the type **without** the `"data-"` prefix
  - e.g. `"data-agent-status"` becomes `byType["agent-status"]`
- `all`: array of the latest value for each type

It keeps only the **latest** part per type based on `timestamp` (or `Date.now()` fallback).

### Transient data parts (out-of-band)

Some streaming payloads may arrive via `useChat`’s `onData` callback without being persisted into `messages`.

This wrapper captures those:

- In `useChat`, `onData` is wrapped.
- If `dataPart.type` starts with `"data-"`, the wrapper updates `_transientDataParts` in the store:
  - if `dataPart.data` is `null` or `undefined`, it removes the entry
  - otherwise it sets `type -> data`

`useDataPart(type)` reads:

1) from messages (latest embedded part)
2) if none found, from `_transientDataParts`

It returns a tuple:

- `[dataOrNull, clear]`
- `clear()` removes the transient value for that type

Notes:

- `useDataParts()` currently reads only from messages, not `_transientDataParts`.
- If you need transient-only values, use `useDataPart()`.

## Extending the store (advanced)

The package is designed to be composable using Zustand’s `StateCreator` pattern.

### Use `createChatStoreCreator` as the base layer

`createChatStoreCreator(initialMessages)` returns a `StateCreator<StoreState<TMessage>>` which:

- installs the canonical fields and actions
- sets up throttling / batching / indexes / memoized selectors
- provides `registerThrottledMessagesEffect(effect)` for derived-cache layers

### Pattern: augment with middleware and derived caches

The example app shows how to create a custom store by composing:

- the base creator
- custom augmenters (higher-order `StateCreator` wrappers)
- Zustand middleware (`subscribeWithSelector`, `devtools`)

Sketch:

```ts
import { createChatStoreCreator } from "@ai-sdk-tools/store";
import { createStore } from "zustand/vanilla";
import { devtools, subscribeWithSelector } from "zustand/middleware";

export function createCustomChatStore(initialMessages = []) {
  return createStore()(
    devtools(
      subscribeWithSelector(
        withMyDerivedCache(
          createChatStoreCreator(initialMessages),
        ),
      ),
      { name: "chat-store" },
    ),
  );
}
```

### When to use `registerThrottledMessagesEffect`

If you maintain a derived cache (e.g. markdown parsing, indexes, analytics), you usually want:

- compute based on `messages`
- update at the same cadence as `_throttledMessages`

`registerThrottledMessagesEffect` lets you attach a callback that runs whenever the throttled snapshot updates.

This is used in `apps/example/src/app/custom-store/with-markdown-memo.ts` to keep a markdown cache in sync with message changes without recomputing on every tiny streaming update more often than necessary.

## Pitfalls and sharp edges

### `useMessageById` throws if missing

`useMessageById(messageId)` throws an error if the message isn’t present.

That’s intentional (it helps catch stale IDs), but in UIs where IDs can race:

- prefer guarding before rendering
- or add your own “maybe” hook using `useChatStore(state => state.getMessageById(id))`

### `useSelector` dependency comparison uses `JSON.stringify`

This is convenient but can be costly or unstable for:

- very large dependency objects
- objects with non-deterministic key order

Prefer primitive deps or small objects.

### Message replacement uses `structuredClone`

`replaceMessage` / `replaceMessageById` call `structuredClone(message)` when writing into the array.

That implies:

- messages should be structured-cloneable (plain JSON-like data is ideal)
- environments must support `structuredClone` (modern browsers / recent Node); if you target older environments, test accordingly

### You still need to think about store identity

If you mount multiple chat instances:

- use separate `<Provider>` instances (and key them if needed)
- avoid accidentally sharing one store across unrelated chats unless that’s what you want

## Debugging and observability

### Debug logging

`debug` is a small logger with configurable levels.

Defaults:

- enabled when `process.env.DEBUG === "true"`
- prefix `"[Store]"`
- level `"warn"`

You can override it at runtime:

```ts
import { configureDebug } from "@ai-sdk-tools/store";

configureDebug({ enabled: true, level: "log" });
```

### Freeze warnings

When enabled, you may see warnings like:

- `[Store] [Freeze] 120ms lastAction= chat:setMessages`

That tells you:

- the main thread blocked long enough to miss frames
- and which store action most recently ran

## Quick internal reference: which hook reads what

- `useChatMessages()` → `state.getThrottledMessages()` (throttled read model)
- `useMessageCount()` → `state.getMessageCount()` (throttled length)
- `useMessageIds()` → `state.getMessageIds()` (throttled IDs)
- `useMessageById(id)` → `state.getMessageById(id)` (Map index; throws if missing)
- `useVirtualMessages(start, end)` → `state.getMessagesSlice(start, end)` (throttled slice)
- `useChatStatus()` / `useChatError()` / `useChatId()` → direct fields
- `useChatActions()` → shallow object containing actions + helper functions (with fallbacks)

## Suggested doc reading order

- **If you’re integrating**: Provider lifecycle → enhanced `useChat` → SSR hydration behavior.
- **If you’re optimizing rendering**: throttled snapshot → O(1) index → selector memoization.
- **If you’re extending**: `createChatStoreCreator` → `registerThrottledMessagesEffect` → custom augmenters.


