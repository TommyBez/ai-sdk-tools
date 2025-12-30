# @ai-sdk-tools/store - Deep Technical Guide

This document provides an in-depth technical exploration of the `@ai-sdk-tools/store` package, covering its architecture, internal mechanisms, performance optimizations, and advanced usage patterns.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Core Data Structures](#core-data-structures)
3. [Performance Optimizations](#performance-optimizations)
4. [Store State & Actions](#store-state--actions)
5. [React Integration & Hooks](#react-integration--hooks)
6. [useChat Synchronization](#usechat-synchronization)
7. [Data Parts System](#data-parts-system)
8. [Debug Utilities](#debug-utilities)
9. [Advanced Customization](#advanced-customization)
10. [TypeScript Support](#typescript-support)
11. [Integration Patterns](#integration-patterns)

---

## Architecture Overview

### Package Structure

```
packages/store/
├── src/
│   ├── index.ts          # Public API exports
│   ├── hooks.ts          # Zustand store & React hooks
│   ├── use-chat.ts       # Enhanced useChat with store sync
│   ├── use-data-parts.ts # Data part extraction hooks
│   └── debug.ts          # Debug logging utilities
├── package.json
└── tsup.config.ts
```

### Design Philosophy

The store package is built around three core principles:

1. **Drop-in Replacement**: Maintain API compatibility with `@ai-sdk/react` while adding performance benefits
2. **Performance First**: Minimize re-renders through intelligent batching, throttling, and memoization
3. **Extensibility**: Enable custom store augmentation through Zustand middleware composition

### Dependency Graph

```
@ai-sdk-tools/store
├── @ai-sdk/react (peer) - Original useChat implementation
├── zustand (peer) - State management foundation
│   ├── zustand/vanilla - Core store creation
│   ├── zustand/middleware - devtools, subscribeWithSelector
│   ├── zustand/shallow - Shallow equality for selectors
│   └── zustand/traditional - useStoreWithEqualityFn
└── ai (dependency) - Core AI types (ChatStatus, etc.)
```

---

## Core Data Structures

### MessageIndex Class

The `MessageIndex` class provides O(1) message lookups via dual hash maps:

```typescript
class MessageIndex<TMessage extends UIMessage> {
  private idToMessage = new Map<string, TMessage>();
  private idToIndex = new Map<string, number>();

  update(messages: TMessage[]) {
    this.idToMessage.clear();
    this.idToIndex.clear();
    messages.forEach((message, index) => {
      this.idToMessage.set(message.id, message);
      this.idToIndex.set(message.id, index);
    });
  }

  getById(id: string): TMessage | undefined {
    return this.idToMessage.get(id);
  }

  getIndexById(id: string): number | undefined {
    return this.idToIndex.get(id);
  }

  has(id: string): boolean {
    return this.idToMessage.has(id);
  }
}
```

**Performance Impact**:
- `getById()`: O(1) vs O(n) for `array.find()`
- `getIndexById()`: O(1) vs O(n) for `array.findIndex()`
- Memory overhead: ~16 bytes per message (two Map entries)

### StoreState Interface

The complete state shape managed by the store:

```typescript
interface StoreState<TMessage extends UIMessage = UIMessage> {
  // Core state
  id: string | undefined;
  messages: TMessage[];
  status: ChatStatus;  // 'ready' | 'streaming' | 'submitted' | 'error'
  error: Error | undefined;

  // Performance optimizations
  _throttledMessages: TMessage[] | null;
  _messageIndex: MessageIndex<TMessage>;
  _memoizedSelectors: Map<string, { result: any; deps: any[] }>;

  // Transient data (not persisted in messages)
  _transientDataParts: Map<string, any>;

  // Actions (detailed below)
  setId, setMessages, setStatus, setError,
  setNewChat, pushMessage, popMessage,
  replaceMessage, replaceMessageById,
  _syncState, reset

  // Chat helpers (synced from useChat)
  sendMessage?, regenerate?, stop?,
  resumeStream?, addToolResult?, clearError?

  // Optimized getters
  getLastMessageId, getMessageIds, getThrottledMessages,
  getInternalMessages, getMessageById, getMessageIndexById,
  getMessagesSlice, getMessageCount, getMemoizedSelector

  // Effects
  registerThrottledMessagesEffect

  // Transient data methods
  setTransientDataPart, getTransientDataPart,
  removeTransientDataPart, clearTransientDataParts
}
```

---

## Performance Optimizations

### 1. Update Batching System

The store implements a priority-based batching system to coalesce multiple updates:

```typescript
const __updateQueue: Array<{ callback: () => void; priority: number }> = [];
let __batchedUpdateScheduled = false;

function batchUpdates(callback: () => void, priority = 0) {
  if (typeof window === "undefined") {
    callback();  // SSR: execute immediately
    return;
  }

  __updateQueue.push({ callback, priority });

  if (!__batchedUpdateScheduled) {
    __batchedUpdateScheduled = true;

    // Use scheduler.postTask if available, otherwise rAF
    const scheduler = (window as any).scheduler;
    const schedule = scheduler?.postTask
      ? scheduler.postTask.bind(scheduler)
      : window.requestAnimationFrame?.bind(window)
        || ((fn: () => void) => setTimeout(fn, 0));

    schedule(() => {
      const updates = __updateQueue.splice(0);
      __batchedUpdateScheduled = false;

      // Sort by priority (higher first) and execute
      updates.sort((a, b) => b.priority - a.priority);
      updates.forEach((update) => update.callback());
    });
  }
}
```

**Priority Levels**:
- `priority: 1` - High priority (streaming updates for smooth text rendering)
- `priority: 0` - Normal priority (default for most operations)

### 2. Enhanced Throttling

Messages updates are throttled at ~60fps for smooth streaming:

```typescript
const MESSAGES_THROTTLE_MS = 16; // ~60fps

function enhancedThrottle<T extends (...args: any[]) => void>(
  func: T,
  wait: number,
): T {
  let timeout: ReturnType<typeof setTimeout> | null = null;
  let previous = 0;
  let pendingArgs: Parameters<T> | null = null;

  const execute = () => {
    if (pendingArgs) {
      func.apply(null, pendingArgs);
      pendingArgs = null;
    }
  };

  return ((...args: Parameters<T>) => {
    const now = Date.now();
    const remaining = wait - (now - previous);
    pendingArgs = args;

    if (remaining <= 0 || remaining > wait) {
      if (timeout) {
        clearTimeout(timeout);
        timeout = null;
      }
      previous = now;

      // Use requestIdleCallback for better performance
      if (typeof window !== "undefined" && (window as any).requestIdleCallback) {
        (window as any).requestIdleCallback(execute, { timeout: 50 });
      } else {
        execute();
      }
    } else if (!timeout) {
      timeout = setTimeout(() => {
        previous = Date.now();
        timeout = null;
        // Same requestIdleCallback optimization
        execute();
      }, remaining);
    }
  }) as T;
}
```

**Key Features**:
- Uses `requestIdleCallback` when available for non-blocking execution
- Falls back to immediate execution when deadline is reached
- Maintains pending arguments to ensure final state is applied

### 3. Throttled Messages Effect System

The store maintains a dual message array system:

```typescript
// Internal state
messages: TMessage[];           // Real-time updates
_throttledMessages: TMessage[]; // Throttled updates for UI

// Effect registration for custom behaviors
registerThrottledMessagesEffect: (effect: () => void) => () => void;
```

Components can register effects that run after throttled updates:

```typescript
// Inside store creator
const throttledEffects = new Set<() => void>();

registerThrottledMessagesEffect: (effect: () => void) => {
  throttledEffects.add(effect);
  return () => {
    throttledEffects.delete(effect);
  };
}

// Executed in throttled updater
throttledEffects.forEach((cb) => {
  try {
    cb();
  } catch (err) {
    console.warn("[chat-store-base] throttled effect error", err);
  }
});
```

### 4. Memoized Selectors

Complex computations are cached with dependency tracking:

```typescript
getMemoizedSelector: <T>(key: string, selector: () => T, deps: any[]): T => {
  const state = get();
  const cached = state._memoizedSelectors.get(key);

  // Fast dependency comparison
  if (
    cached &&
    cached.deps.length === deps.length &&
    (deps.length === 0 || JSON.stringify(cached.deps) === JSON.stringify(deps))
  ) {
    return cached.result;
  }

  const result = selector();
  state._memoizedSelectors.set(key, { result, deps: [...deps] });
  return result;
}
```

**Usage Example**:

```typescript
const userMessageCount = useSelector(
  'userMessages',
  (messages) => messages.filter(m => m.role === 'user').length,
  [messages.length]
);
```

### 5. Freeze Detector

Development-time performance monitoring:

```typescript
function startFreezeDetector({ thresholdMs = 80 } = {}): void {
  if (typeof window === "undefined" || __freezeDetectorStarted) return;

  const tick = (now: number) => {
    const expected = __freezeLastTs + 16.7;  // 60fps target
    const blockedMs = now - expected;
    if (blockedMs > thresholdMs) {
      debug.warn(
        "[Freeze]",
        `${Math.round(blockedMs)}ms`,
        "lastAction=",
        __lastActionLabel,
      );
    }
    __freezeLastTs = now;
    __freezeRafId = window.requestAnimationFrame(tick);
  };

  __freezeRafId = window.requestAnimationFrame(tick);
}
```

This helps identify performance bottlenecks during development by logging when the main thread is blocked.

---

## Store State & Actions

### Action Implementations

All actions use the batching system with labeled markers for debugging:

#### setMessages

```typescript
setMessages: (messages) => {
  markLastAction("chat:setMessages");
  batchUpdates(() => {
    const currentState = get();
    if (messages === currentState.messages) return;  // Reference equality check

    set({
      messages: messages,
      _memoizedSelectors: new Map(),  // Clear cache
    });

    // During streaming, update immediately for smooth rendering
    if (currentState.status === "streaming") {
      batchUpdates(() => {
        const state = get();
        const newThrottledMessages = [...state.messages];
        state._messageIndex.update(newThrottledMessages);
        set({ _throttledMessages: newThrottledMessages });
      }, 1);  // High priority
    } else {
      throttledMessagesUpdater?.();
    }
  });
}
```

#### replaceMessageById

O(1) message replacement using the index:

```typescript
replaceMessageById: (id, message) => {
  markLastAction("chat:replaceMessageById");
  batchUpdates(() => {
    const currentState = get();
    set((state) => {
      const index = state._messageIndex.getIndexById(id);
      if (index === undefined) return state;

      const newMessages = [...state.messages];
      newMessages[index] = structuredClone(message);  // Deep clone
      return {
        messages: newMessages,
        _memoizedSelectors: new Map(),
      };
    });
    // ... streaming optimization
  });
}
```

#### reset

Complete state reset with cleanup:

```typescript
reset: () => {
  markLastAction("chat:reset");
  batchUpdates(() => {
    const state = get();
    const newMessageIndex = new MessageIndex<TMessage>();
    newMessageIndex.update([]);

    // Sync with chat helpers if available
    if (state.setMessages) {
      state.setMessages([]);
    }

    set({
      id: undefined,
      messages: [],
      status: "ready" as const,
      error: undefined,
      _throttledMessages: [],
      _messageIndex: newMessageIndex,
      _memoizedSelectors: new Map(),
      _transientDataParts: new Map(),
    });
  });
}
```

---

## React Integration & Hooks

### Provider Component

The Provider initializes or accepts an external store:

```typescript
export function Provider<TMessage extends UIMessage = UIMessage>({
  children,
  initialMessages,
  store,
}: {
  children: React.ReactNode;
  initialMessages?: TMessage[];
  store?: CompatibleChatStoreApi<TMessage>;
}) {
  const storeRef = useRef<CompatibleChatStoreApi<TMessage> | null>(null);

  if (storeRef.current === null) {
    storeRef.current = store || createChatStore<TMessage>(initialMessages || []);
  }

  return React.createElement(
    ChatStoreContext.Provider,
    { value: storeRef.current },
    children,
  );
}
```

### Hook Architecture

The hooks follow a layered architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Application Components                      │
├─────────────────────────────────────────────────────────────────┤
│  useChatMessages  │  useChatStatus  │  useMessageById  │ etc.   │
├─────────────────────────────────────────────────────────────────┤
│                         useChatStore                             │
├─────────────────────────────────────────────────────────────────┤
│            useStore (Zustand)  ←  ChatStoreContext              │
├─────────────────────────────────────────────────────────────────┤
│                    createChatStore (Zustand)                     │
└─────────────────────────────────────────────────────────────────┘
```

### Stable Selector Pattern

Selectors are defined outside components to prevent recreation:

```typescript
// Stable selector functions
const statusSelector = (state: StoreState<any>) => state.status;
const errorSelector = (state: StoreState<any>) => state.error;
const idSelector = (state: StoreState<any>) => state.id;
const messageCountSelector = (state: StoreState<any>) => state.getMessageCount();

// Hook implementations
export const useChatStatus = () => useChatStore(statusSelector);
export const useChatError = () => useChatStore(errorSelector);
export const useChatId = () => useChatStore(idSelector);
export const useMessageCount = () => useChatStore(messageCountSelector);
```

### useShallow Integration

For array/object selections that need shallow equality:

```typescript
export const useChatMessages = <TMessage extends UIMessage = UIMessage>() => {
  return useChatStore(
    useShallow((state: StoreState<TMessage>) => state.getThrottledMessages()),
  );
};

export const useMessageIds = <TMessage extends UIMessage = UIMessage>() =>
  useChatStore(
    useShallow((state: StoreState<TMessage>) => state.getMessageIds()),
  );
```

### Virtualization Hook

For large message lists:

```typescript
export const useVirtualMessages = <TMessage extends UIMessage = UIMessage>(
  start: number,
  end?: number,
) => {
  return useChatStore(
    useCallback(
      (state: StoreState<TMessage>) => state.getMessagesSlice(start, end),
      [start, end],
    ),
  );
};
```

---

## useChat Synchronization

### Sync Architecture

The enhanced `useChat` hook syncs state bidirectionally:

```typescript
export function useChat<TMessage extends UIMessage = UIMessage>(
  options: UseChatOptionsWithPerformance<TMessage> = {},
): UseChatHelpers<TMessage> {
  const {
    store: customStore,
    enableBatching = true,
    ...originalOptions
  } = options;

  // Use custom or context store
  const contextStore = useChatStoreApi<TMessage>();
  const store = customStore || contextStore;

  // Original AI SDK hook
  const chatHelpers = useOriginalChat<TMessage>({
    ...originalOptions,
    onData: wrappedOnData,  // Captures transient data parts
  });

  // Sync effect
  useEffect(() => {
    const currentStoreState = (store as any).getState?.() || { messages: [] };

    // Preserve server-side messages during hydration
    const shouldSyncMessages = !(
      currentStoreState.messages?.length > 0 &&
      chatHelpers.messages.length === 0
    );

    const stateData: any = {
      id: chatHelpers.id,
      error: chatHelpers.error,
      status: chatHelpers.status,
    };

    if (shouldSyncMessages) {
      stateData.messages = chatHelpers.messages;
    }

    // Sync functions
    const functionsData = {
      sendMessage: chatHelpers.sendMessage,
      regenerate: chatHelpers.regenerate,
      stop: chatHelpers.stop,
      resumeStream: chatHelpers.resumeStream,
      addToolResult: chatHelpers.addToolResult,
      setMessages: chatHelpers.setMessages,
      clearError: chatHelpers.clearError,
    };

    const chatState = { ...stateData, ...functionsData };

    if (enableBatching && typeof window !== "undefined") {
      window.requestAnimationFrame(() => syncState(chatState));
    } else {
      syncState(chatState);
    }
  }, [/* dependencies */]);

  // Return store messages as source of truth
  const storeMessages = useStore(
    store as any,
    (state: any) => state.messages as TMessage[],
  );

  return {
    ...chatHelpers,
    messages: storeMessages || chatHelpers.messages,
  };
}
```

### Hydration Protection

The sync logic protects server-side rendered messages:

```typescript
// Skip syncing messages if store has messages but chat doesn't
// This prevents clearing server-side messages on hydration
const shouldSyncMessages = !(
  currentStoreState.messages?.length > 0 &&
  chatHelpers.messages.length === 0
);
```

---

## Data Parts System

### Transient Data Parts

Data parts that exist during streaming but aren't persisted in messages:

```typescript
// Store methods
setTransientDataPart: (type, data) => {
  markLastAction("chat:setTransientDataPart");
  batchUpdates(() => {
    set((state) => {
      const newTransientDataParts = new Map(state._transientDataParts);
      newTransientDataParts.set(type, data);
      return { _transientDataParts: newTransientDataParts };
    });
  });
}

getTransientDataPart: (type) => {
  const state = get();
  return state._transientDataParts.get(type);
}

removeTransientDataPart: (type) => {
  // ... implementation
}
```

### useDataParts Hook

Extracts all data parts from messages:

```typescript
export function useDataParts(): UseDataPartsReturn {
  const messages = useChatMessages();

  return useMemo(() => {
    const dataParts = extractDataPartsFromMessages(messages);

    // Group by type (strip 'data-' prefix)
    const byType: Record<string, DataPart<unknown>> = {};

    for (const dataPart of dataParts) {
      const key = dataPart.type.replace(/^data-/, "");
      const existing = byType[key];
      if (
        !existing ||
        (dataPart.timestamp && existing.timestamp &&
          dataPart.timestamp > existing.timestamp)
      ) {
        byType[key] = dataPart;
      }
    }

    return {
      byType,
      all: Object.values(byType),
    };
  }, [messages]);
}
```

### useDataPart Hook

Get specific data part with useState-like API:

```typescript
export function useDataPart<T = unknown>(
  type: string,
  options?: UseDataPartOptions<T>,
): [T | null, () => void] {
  const messages = useChatMessages();
  const { onData } = options || {};

  // Subscribe to transient data
  const transientDataParts = useChatStore((state) => state._transientDataParts);
  const removeTransientDataPart = useChatStore((state) => state.removeTransientDataPart);

  const result = useMemo(() => {
    const dataParts = extractDataPartsFromMessages(messages);
    const fullType = type.startsWith("data-") ? type : `data-${type}`;

    // Search in message parts
    let latest: DataPart<T> | null = null;
    for (const dataPart of dataParts) {
      if (dataPart.type === fullType) {
        if (!latest || dataPart.timestamp > latest.timestamp) {
          latest = dataPart as DataPart<T>;
        }
      }
    }

    // Fallback to transient data
    if (!latest) {
      const transientData = transientDataParts.get(fullType);
      if (transientData !== undefined) {
        latest = { type: fullType, data: transientData };
      }
    }

    return latest;
  }, [messages, type, transientDataParts]);

  // onData callback
  useEffect(() => {
    if (result && onData) {
      onData(result);
    }
  }, [result, onData]);

  // Clear function
  const clear = useCallback(() => {
    const fullType = type.startsWith("data-") ? type : `data-${type}`;
    removeTransientDataPart(fullType);
  }, [type, removeTransientDataPart]);

  return [result ? result.data : null, clear];
}
```

### Data Part Extraction

Recursively extracts data parts from message parts:

```typescript
function extractDataPartsFromMessages(messages: UIMessage[]): DataPart<unknown>[] {
  const dataParts: DataPart<unknown>[] = [];

  for (const message of messages) {
    if (message.parts && Array.isArray(message.parts)) {
      for (const part of message.parts) {
        // Direct data parts
        if (part.type.startsWith("data-") && "data" in part) {
          dataParts.push({
            type: part.type,
            data: part.data,
            timestamp: part.timestamp || Date.now(),
          });
        }

        // Nested in tool call results
        if (part.type.startsWith("tool-") && "result" in part && part.result) {
          const result = part.result;
          if (typeof result === "object" && result && "parts" in result) {
            const parts = (result as { parts?: unknown[] }).parts;
            if (Array.isArray(parts)) {
              for (const nestedPart of parts) {
                if (nestedPart.type?.startsWith("data-") && nestedPart.data !== undefined) {
                  dataParts.push({
                    type: nestedPart.type,
                    data: nestedPart.data,
                    timestamp: nestedPart.timestamp || Date.now(),
                  });
                }
              }
            }
          }
        }
      }
    }
  }

  return dataParts;
}
```

---

## Debug Utilities

### DebugLogger Class

```typescript
type LogLevel = "log" | "warn" | "error";

interface DebugOptions {
  enabled?: boolean;
  prefix?: string;
  level?: LogLevel;
}

class DebugLogger {
  private enabled: boolean;
  private prefix: string;
  private level: LogLevel;

  constructor(options: DebugOptions = {}) {
    this.enabled = options.enabled ?? process.env.DEBUG === "true";
    this.prefix = options.prefix ?? "[Store]";
    this.level = options.level ?? "warn";
  }

  private shouldLog(level: LogLevel): boolean {
    if (!this.enabled) return false;
    const levels = ["log", "warn", "error"];
    return levels.indexOf(level) >= levels.indexOf(this.level);
  }

  log(...args: any[]): void {
    if (this.shouldLog("log")) {
      console.log(this.prefix, ...args);
    }
  }

  warn(...args: any[]): void {
    if (this.shouldLog("warn")) {
      console.warn(this.prefix, ...args);
    }
  }

  error(...args: any[]): void {
    if (this.shouldLog("error")) {
      console.error(this.prefix, ...args);
    }
  }
}

// Default instance
export const debug = new DebugLogger();

// Configuration function
export function configureDebug(options: DebugOptions): void {
  if (options.enabled !== undefined) {
    debug.setEnabled(options.enabled);
  }
  if (options.level !== undefined) {
    debug.setLevel(options.level);
  }
}
```

### Enabling Debug Mode

```bash
# Environment variable
DEBUG=true npm run dev

# Programmatic
import { configureDebug } from '@ai-sdk-tools/store';
configureDebug({ enabled: true, level: 'log' });
```

---

## Advanced Customization

### Middleware Composition Pattern

The store supports Zustand middleware composition for extension:

```typescript
import {
  createChatStoreCreator,
  ChatStoreContext,
  Provider as ChatProvider,
} from "@ai-sdk-tools/store";
import { devtools, subscribeWithSelector } from "zustand/middleware";
import { createStore } from "zustand/vanilla";

// Custom state augmentation
interface CustomState<TMessage extends UIMessage> extends StoreState<TMessage> {
  customField: string;
  customAction: () => void;
}

// Middleware that adds custom functionality
const withCustomFeature = <TMessage extends UIMessage>(
  creator: StateCreator<StoreState<TMessage>, [], []>
): StateCreator<CustomState<TMessage>, [], []> => (set, get, api) => {
  const base = creator(set, get, api);
  return {
    ...base,
    customField: '',
    customAction: () => {
      set({ customField: 'updated' } as Partial<CustomState<TMessage>>);
    },
  };
};

// Create custom store
function createCustomStore<TMessage extends UIMessage>(initialMessages: TMessage[] = []) {
  return createStore<CustomState<TMessage>>()(
    devtools(
      subscribeWithSelector(
        withCustomFeature(createChatStoreCreator<TMessage>(initialMessages))
      ),
      { name: "custom-chat-store" }
    )
  );
}

// Custom provider
export function CustomProvider({ children, initialMessages }) {
  const storeRef = useRef(null);
  if (storeRef.current === null) {
    storeRef.current = createCustomStore(initialMessages);
  }
  return (
    <ChatProvider store={storeRef.current}>
      {children}
    </ChatProvider>
  );
}
```

### Message Parts Extension

Example from the example app - adding typed message part access:

```typescript
export type PartsAugmentedState<UM extends UIMessage> = StoreState<UM> & {
  getMessagePartTypesById: (messageId: string) => PartType[];
  getMessagePartsRangeCached: (
    messageId: string,
    startIdx: number,
    endIdx: number,
    type?: string,
  ) => Parts;
  getMessagePartByIdxCached: (messageId: string, partIdx: number) => Part;
};

export const withMessageParts = <UI_MESSAGE extends UIMessage, T extends StoreState<UI_MESSAGE>>(
  creator: StateCreator<T, [], []>,
): StateCreator<T & PartsAugmentedState<UI_MESSAGE>, [], []> => (set, get, api) => {
  const base = creator(set, get, api);
  return {
    ...base,

    getMessagePartTypesById: (messageId: string) => {
      const state = get();
      const message = (state._throttledMessages || state.messages).find(
        (msg) => msg.id === messageId,
      );
      if (!message) throw new Error(`Message not found: ${messageId}`);
      return message.parts.map((p) => p.type);
    },

    getMessagePartsRangeCached: (messageId, startIdx, endIdx, type) => {
      // ... implementation
    },

    getMessagePartByIdxCached: (messageId, partIdx) => {
      // ... implementation
    },
  };
};
```

### Markdown Caching Extension

Pre-compute and cache markdown parsing:

```typescript
export interface MarkdownMemoAugmentedState<UI_MESSAGE extends UIMessage>
  extends StoreState<UI_MESSAGE> {
  _markdownCache: Map<string, MarkdownCacheEntry>;
  getMarkdownBlocksForPart: (messageId: string, partIdx: number) => string[];
  getMarkdownBlockCountForPart: (messageId: string, partIdx: number) => number;
  getMarkdownBlockByIndex: (messageId: string, partIdx: number, blockIdx: number) => string | null;
}

export const withMarkdownMemo = <UI_MESSAGE extends UIMessage>(
  initialMessages: UI_MESSAGE[] = [],
) => <T extends StoreState<UI_MESSAGE>>(
  creator: StateCreator<T, [], []>,
): StateCreator<T & MarkdownMemoAugmentedState<UI_MESSAGE>, [], []> => (set, get, api) => {
  const initialPrecompute = precomputeMarkdownForAllMessages(initialMessages);
  const base = creator(set, get, api);

  // Register effect for automatic cache updates
  base.registerThrottledMessagesEffect(() => {
    const state = get();
    const { cache } = precomputeMarkdownForAllMessages(
      state.messages,
      get()._markdownCache,
    );
    set({ _markdownCache: cache } as Partial<...>);
  });

  return {
    ...base,
    _markdownCache: initialPrecompute.cache,
    getMarkdownBlocksForPart: (messageId, partIdx) => {
      // ... implementation
    },
    // ... other methods
  };
};
```

### Composing Multiple Extensions

```typescript
function createFullyAugmentedStore<TMessage extends UIMessage>(
  initialMessages: TMessage[] = [],
) {
  return createStore<CustomState<TMessage>>()(
    devtools(
      subscribeWithSelector(
        withMarkdownMemo<TMessage>(initialMessages)(
          withMessageParts(
            withCustomFeature(
              createChatStoreCreator<TMessage>(initialMessages)
            )
          )
        )
      ),
      { name: "augmented-chat-store" }
    )
  );
}
```

---

## TypeScript Support

### Generic Message Types

The store fully supports custom message types:

```typescript
import type { UIMessage } from "@ai-sdk/react";

// Custom message with metadata
interface CustomMessage extends UIMessage {
  metadata?: {
    userId: string;
    timestamp: number;
    source: 'api' | 'user';
  };
  customData?: {
    sentiment?: 'positive' | 'negative' | 'neutral';
    tokens?: number;
  };
}

// Usage
const chat = useChat<CustomMessage>({
  transport: new DefaultChatTransport({ api: '/api/chat' })
});

const messages = useChatMessages<CustomMessage>();

const message = useMessageById<CustomMessage>(messageId);
```

### Type-safe Hooks

Custom hooks with proper typing:

```typescript
// Custom hook with full type inference
function useTypedChatStore<T, TMessage extends UIMessage = UIMessage>(
  selector: (store: StoreState<TMessage>) => T,
): T {
  return useChatStore(selector);
}

// Usage
const status = useTypedChatStore((state) => state.status);
// status is inferred as ChatStatus
```

### ChatActions Type

```typescript
export type ChatActions<TMessage extends UIMessage = UIMessage> = {
  setMessages: (messages: TMessage[]) => void;
  pushMessage: (message: TMessage) => void;
  popMessage: () => void;
  replaceMessage: (index: number, message: TMessage) => void;
  replaceMessageById: (id: string, message: TMessage) => void;
  setStatus: (status: ChatStatus) => void;
  setError: (error: Error | undefined) => void;
  setId: (id: string | undefined) => void;
  setNewChat: (id: string, messages: TMessage[]) => void;
  reset: () => void;
  sendMessage: UseChatHelpers<TMessage>["sendMessage"];
  regenerate: UseChatHelpers<TMessage>["regenerate"];
  stop: UseChatHelpers<TMessage>["stop"];
  resumeStream: UseChatHelpers<TMessage>["resumeStream"];
  addToolResult: UseChatHelpers<TMessage>["addToolResult"];
  clearError: UseChatHelpers<TMessage>["clearError"];
};
```

---

## Integration Patterns

### With DevTools

The store integrates with `@ai-sdk-tools/devtools`:

```typescript
// Store automatically includes devtools middleware
createStore<StoreState<TMessage>>()(
  devtools(
    subscribeWithSelector(createChatStoreCreator<TMessage>(initialMessages)),
    { name: "chat-store" },
  ),
);
```

### Multiple Chat Instances

```typescript
function MultiChatApp() {
  return (
    <>
      <Provider key="chat-1">
        <ChatWindow id="1" />
      </Provider>
      <Provider key="chat-2">
        <ChatWindow id="2" />
      </Provider>
    </>
  );
}
```

### Server-Side Rendering

The store handles SSR gracefully:

```typescript
// Batching falls back to synchronous on server
function batchUpdates(callback: () => void, priority = 0) {
  if (typeof window === "undefined") {
    callback();  // SSR: execute immediately
    return;
  }
  // Client-side batching...
}

// Provider preserves store reference
const storeRef = useRef<ChatStoreApi | null>(null);
if (storeRef.current === null) {
  storeRef.current = store || createChatStore(initialMessages || []);
}
```

### Persisting Chat History

```typescript
function usePersistentChat() {
  const { messages, setMessages } = useChatStore((state) => ({
    messages: state.messages,
    setMessages: state.setMessages,
  }));

  // Save to localStorage
  useEffect(() => {
    if (messages.length > 0) {
      localStorage.setItem('chat-history', JSON.stringify(messages));
    }
  }, [messages]);

  // Restore on mount
  useEffect(() => {
    const saved = localStorage.getItem('chat-history');
    if (saved) {
      try {
        const parsed = JSON.parse(saved);
        setMessages(parsed);
      } catch (e) {
        console.error('Failed to restore chat history');
      }
    }
  }, []);

  return { messages };
}
```

---

## Performance Best Practices

### 1. Use Specific Selectors

```typescript
// ❌ Bad: Re-renders on any state change
const state = useChatStore();

// ✅ Good: Only re-renders when status changes
const status = useChatStatus();
```

### 2. Leverage Message Index

```typescript
// ❌ Bad: O(n) lookup
const message = messages.find(m => m.id === id);

// ✅ Good: O(1) lookup
const message = useMessageById(id);
```

### 3. Use Virtualization for Large Lists

```typescript
// ❌ Bad: Renders all messages
const messages = useChatMessages();

// ✅ Good: Only renders visible messages
const visibleMessages = useVirtualMessages(startIndex, endIndex);
```

### 4. Memoize Computed Values

```typescript
// ❌ Bad: Recomputes on every render
const userMessages = messages.filter(m => m.role === 'user');

// ✅ Good: Cached until dependencies change
const userMessages = useSelector(
  'userMessages',
  (msgs) => msgs.filter(m => m.role === 'user'),
  [messages.length]
);
```

### 5. Avoid Inline Selectors

```typescript
// ❌ Bad: Creates new function every render
const status = useChatStore((state) => state.status);

// ✅ Good: Stable selector reference
const statusSelector = (state) => state.status;
const status = useChatStore(statusSelector);

// Or use the provided hook
const status = useChatStatus();
```

---

## Troubleshooting

### Common Issues

**1. "useChatStore must be used within Provider"**

Ensure your component tree is wrapped with `<Provider>`:

```tsx
<Provider initialMessages={[]}>
  <YourApp />
</Provider>
```

**2. Messages not updating during streaming**

Check that `status` is correctly set to `'streaming'`. The store uses status to determine update priority.

**3. Memory leaks with effects**

Always clean up registered effects:

```typescript
useEffect(() => {
  const cleanup = store.getState().registerThrottledMessagesEffect(myEffect);
  return cleanup;  // Important!
}, []);
```

**4. Type errors with custom messages**

Ensure your custom type extends `UIMessage`:

```typescript
interface MyMessage extends UIMessage {
  // your fields
}
```

### Debug Checklist

1. Enable debug logging: `DEBUG=true`
2. Check freeze detector warnings for performance issues
3. Verify store is connected via React DevTools
4. Inspect Zustand DevTools for state changes

---

## Deep Dive: initialMessages and sendMessage Interaction

This section provides an in-depth exploration of how `initialMessages` flows through the system and interacts with the `sendMessage` function from the `useChat` hook.

### The Complete Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SERVER SIDE                                     │
│                                                                              │
│  1. Page Component (SSR)                                                    │
│     └── loadChatHistory(chatId) → fetches messages from database/memory     │
│         └── returns UIMessage[]                                             │
│                                                                              │
│  2. initialMessages passed to Provider                                       │
│     └── <Provider initialMessages={initialMessages}>                         │
└────────────────────────────────────────────────────────────────┬────────────┘
                                                                 │
                                                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT SIDE                                     │
│                                                                              │
│  3. Store Creation (runs once on mount)                                      │
│     └── createChatStore(initialMessages)                                     │
│         ├── messages: initialMessages                                        │
│         ├── _throttledMessages: [...initialMessages]                         │
│         └── _messageIndex.update(initialMessages)                            │
│                                                                              │
│  4. useChat Hook Initialization                                              │
│     └── useOriginalChat() → AI SDK's useChat                                │
│         └── starts with empty messages (no initialMessages passed)          │
│                                                                              │
│  5. Sync Effect (useChat → Store)                                            │
│     └── Detects: store has messages, chatHelpers has none                   │
│     └── SKIPS syncing messages (hydration protection)                        │
│     └── Only syncs: id, status, error, functions                            │
│                                                                              │
│  6. Component Renders                                                        │
│     └── useChat returns store.messages (not chatHelpers.messages)           │
│     └── UI shows initialMessages immediately                                 │
│                                                                              │
│  7. User calls sendMessage()                                                 │
│     └── chatHelpers.sendMessage() → AI SDK processes                        │
│     └── AI SDK updates chatHelpers.messages                                 │
│     └── Sync effect runs: syncs new messages to store                        │
│     └── Store updates → Components re-render                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step: initialMessages Flow

#### Step 1: Server-Side Data Loading

In a Next.js App Router setup, messages are loaded server-side:

```typescript
// app/[[...chatId]]/page.tsx
import { loadChatHistory } from "@/lib/data";

export default async function Page({ params }: Props) {
  const { chatId } = await params;
  const currentChatId = chatId?.at(0);

  // Server-side: Load messages from database/memory
  const initialMessages = currentChatId
    ? await loadChatHistory(currentChatId)
    : [];

  return (
    <CustomStoreProvider
      chatKey={currentChatId || "home"}
      initialMessages={initialMessages}
    >
      <ChatInterface />
    </CustomStoreProvider>
  );
}
```

The data loading function typically fetches from a persistence layer:

```typescript
// lib/data.ts
export async function loadChatHistory(chatId: string): Promise<UIMessage[]> {
  try {
    const messages = await memoryProvider.getMessages({
      chatId,
      limit: 50,
    });
    return messages || [];
  } catch (error) {
    console.error("Error loading chat history:", error);
    return [];
  }
}
```

#### Step 2: Provider Receives initialMessages

The Provider creates the store with initial messages:

```typescript
// Provider implementation
export function Provider<TMessage extends UIMessage = UIMessage>({
  children,
  initialMessages,
  store,
}: {
  children: React.ReactNode;
  initialMessages?: TMessage[];
  store?: CompatibleChatStoreApi<TMessage>;
}) {
  const storeRef = useRef<CompatibleChatStoreApi<TMessage> | null>(null);

  // Store is created ONCE on first render
  if (storeRef.current === null) {
    storeRef.current =
      store || createChatStore<TMessage>(initialMessages || []);
  }

  return React.createElement(
    ChatStoreContext.Provider,
    { value: storeRef.current },
    children,
  );
}
```

**Important**: The store reference is created once and never recreated, even if `initialMessages` prop changes. This is intentional for performance.

#### Step 3: Store Initialization

When `createChatStore` is called, it initializes all internal state:

```typescript
export function createChatStoreCreator<TMessage extends UIMessage>(
  initialMessages: TMessage[] = [],
): StateCreator<StoreState<TMessage>, [], []> {
  const messageIndex = new MessageIndex<TMessage>();

  // Index is pre-populated for O(1) lookups
  messageIndex.update(initialMessages);

  return (set, get) => {
    return {
      id: undefined,
      messages: initialMessages,              // Direct reference to initial messages
      status: "ready" as const,
      error: undefined,
      _throttledMessages: [...initialMessages], // Shallow copy for throttled updates
      _messageIndex: messageIndex,              // Pre-built index
      _memoizedSelectors: new Map(),
      _transientDataParts: new Map(),
      // ... actions and helpers
    };
  };
}
```

**Key Points**:
- `messages` holds the direct reference to `initialMessages`
- `_throttledMessages` is a shallow copy (for independent throttling)
- `_messageIndex` is pre-built for immediate O(1) lookups
- Status starts as `"ready"` (not streaming)

#### Step 4: useChat Hook Synchronization

The enhanced `useChat` hook manages bidirectional sync:

```typescript
export function useChat<TMessage extends UIMessage = UIMessage>(
  options: UseChatOptionsWithPerformance<TMessage> = {},
): UseChatHelpers<TMessage> {
  const { store: customStore, enableBatching = true, ...originalOptions } = options;

  const contextStore = useChatStoreApi<TMessage>();
  const store = customStore || contextStore;

  // AI SDK's original useChat - starts with EMPTY messages
  const chatHelpers = useOriginalChat<TMessage>({
    ...originalOptions,
    onData: wrappedOnData,
  });

  // Sync effect - runs when chatHelpers state changes
  useEffect(() => {
    const currentStoreState = (store as any).getState?.() || { messages: [] };

    // CRITICAL: Hydration protection
    // If store has messages but chatHelpers doesn't, DON'T overwrite store
    const shouldSyncMessages = !(
      currentStoreState.messages?.length > 0 &&
      chatHelpers.messages.length === 0
    );

    const stateData: any = {
      id: chatHelpers.id,
      error: chatHelpers.error,
      status: chatHelpers.status,
    };

    // Only sync messages if appropriate
    if (shouldSyncMessages) {
      stateData.messages = chatHelpers.messages;
    }

    // Always sync functions (sendMessage, regenerate, etc.)
    const functionsData = {
      sendMessage: chatHelpers.sendMessage,
      regenerate: chatHelpers.regenerate,
      stop: chatHelpers.stop,
      resumeStream: chatHelpers.resumeStream,
      addToolResult: chatHelpers.addToolResult,
      setMessages: chatHelpers.setMessages,
      clearError: chatHelpers.clearError,
    };

    syncState({ ...stateData, ...functionsData });
  }, [/* dependencies */]);

  // CRITICAL: Return store messages, not chatHelpers messages
  const storeMessages = useStore(
    store as any,
    (state: any) => state.messages as TMessage[],
  );

  return {
    ...chatHelpers,
    messages: storeMessages || chatHelpers.messages,
  };
}
```

**The Hydration Protection Logic**:

```typescript
const shouldSyncMessages = !(
  currentStoreState.messages?.length > 0 &&  // Store has messages (from SSR)
  chatHelpers.messages.length === 0           // AI SDK has none (just initialized)
);
```

This prevents the scenario where:
1. Server renders page with 10 messages in store
2. Client hydrates, AI SDK's useChat starts with 0 messages
3. Without protection: store would be cleared to 0 messages
4. With protection: store keeps its 10 messages

### Step-by-Step: sendMessage Flow

When a user sends a message, here's the complete flow:

#### Step 1: Component Calls sendMessage

```typescript
// In a component
const { sendMessage, status } = useChat({
  id: chatId,
  transport: new DefaultChatTransport({ api: "/api/chat" }),
});

const handleSubmit = (message: ChatInputMessage) => {
  sendMessage({
    text: message.text,
    files: message.files,
    metadata: { agentChoice: message.metadata?.agentChoice },
  });
};
```

#### Step 2: sendMessage Execution Path

```
sendMessage() called
    │
    ▼
Store's sendMessage (synced from chatHelpers)
    │
    ▼
AI SDK's chatHelpers.sendMessage()
    │
    ├── Creates user message with generated ID
    ├── Adds to internal messages array
    ├── Sets status to "submitted"
    ├── Initiates API request via transport
    │
    ▼
API streaming begins
    │
    ├── Status changes to "streaming"
    ├── Assistant message created (empty)
    ├── Content streams in, message updated
    │
    ▼
Sync effect triggers (chatHelpers state changed)
    │
    ├── shouldSyncMessages = true (both have messages now)
    ├── Syncs messages to store
    ├── Store's setMessages() called
    │
    ▼
Store updates
    │
    ├── messages array updated
    ├── _memoizedSelectors cleared
    ├── During streaming: immediate _throttledMessages update (high priority)
    ├── Not streaming: throttled update (~16ms)
    │
    ▼
_messageIndex rebuilt
    │
    ▼
Components re-render via Zustand subscriptions
```

#### Step 3: Message Flow During Streaming

During streaming, the store handles high-frequency updates efficiently:

```typescript
setMessages: (messages) => {
  markLastAction("chat:setMessages");
  batchUpdates(() => {
    const currentState = get();
    if (messages === currentState.messages) return;  // Skip if same reference

    set({
      messages: messages,
      _memoizedSelectors: new Map(),  // Clear computed cache
    });

    // STREAMING OPTIMIZATION
    if (currentState.status === "streaming") {
      // Immediate update for smooth text rendering
      batchUpdates(() => {
        const state = get();
        const newThrottledMessages = [...state.messages];
        state._messageIndex.update(newThrottledMessages);
        set({ _throttledMessages: newThrottledMessages });
      }, 1);  // Priority 1 = high priority
    } else {
      // Normal: use throttled updater (~60fps)
      throttledMessagesUpdater?.();
    }
  });
}
```

### Real-World Example: Complete Chat Implementation

Here's a complete example showing initialMessages and sendMessage working together:

```typescript
// 1. Server Component - Load and provide initial messages
// app/chat/[chatId]/page.tsx
import { Provider } from "@ai-sdk-tools/store";

export default async function ChatPage({ params }: { params: { chatId: string } }) {
  // Server-side data fetching
  const initialMessages = await db.messages.findMany({
    where: { chatId: params.chatId },
    orderBy: { createdAt: 'asc' },
  });

  return (
    <Provider initialMessages={initialMessages}>
      <ChatContainer chatId={params.chatId} />
    </Provider>
  );
}

// 2. Client Component - Use chat with sendMessage
// components/chat-container.tsx
"use client";

import { useChat, useChatMessages, useChatStatus } from "@ai-sdk-tools/store";
import { DefaultChatTransport } from "ai";

export function ChatContainer({ chatId }: { chatId: string }) {
  const [input, setInput] = useState("");
  
  // useChat returns store messages (including initialMessages)
  const { sendMessage, stop } = useChat({
    id: chatId,
    transport: new DefaultChatTransport({
      api: "/api/chat",
      prepareSendMessagesRequest({ messages, id }) {
        return {
          body: {
            messages,
            chatId: id,
          },
        };
      },
    }),
  });

  // These hooks also work - they read from the same store
  const messages = useChatMessages();
  const status = useChatStatus();

  const handleSend = () => {
    if (!input.trim()) return;
    
    sendMessage({
      text: input,
      metadata: { timestamp: Date.now() },
    });
    
    setInput("");
  };

  return (
    <div>
      {/* Messages include initialMessages + new messages */}
      <div className="messages">
        {messages.map((msg) => (
          <MessageBubble key={msg.id} message={msg} />
        ))}
      </div>
      
      <div className="input-area">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && handleSend()}
          disabled={status === 'streaming'}
        />
        <button onClick={status === 'streaming' ? stop : handleSend}>
          {status === 'streaming' ? 'Stop' : 'Send'}
        </button>
      </div>
    </div>
  );
}

// 3. Separate component accessing same messages (no prop drilling!)
// components/message-count.tsx
"use client";

import { useMessageCount } from "@ai-sdk-tools/store";

export function MessageCount() {
  // This includes initialMessages count
  const count = useMessageCount();
  return <span>{count} messages</span>;
}
```

### Timing Diagram: Hydration with initialMessages

```
Timeline: SSR → Hydration → User Interaction

SERVER (SSR):
────────────────────────────────────────────────────────────────
t0: Page request received
t1: loadChatHistory() fetches 5 messages from database
t2: Provider created with initialMessages=[msg1,msg2,msg3,msg4,msg5]
t3: HTML rendered with 5 messages
t4: HTML sent to client

CLIENT (Hydration):
────────────────────────────────────────────────────────────────
t5: HTML received, initial paint shows 5 messages
t6: React hydration begins
t7: Provider mounts
    - storeRef.current = createChatStore([msg1,msg2,msg3,msg4,msg5])
    - Store state: { messages: [5 messages], status: 'ready' }
t8: ChatContainer mounts
t9: useChat() called
    - useOriginalChat() initializes with messages: []
    - chatHelpers = { messages: [], status: 'ready', ... }
t10: useEffect sync runs
    - Store has 5 messages, chatHelpers has 0
    - shouldSyncMessages = false (hydration protection!)
    - Only syncs: id, status, error, functions
    - Store keeps its 5 messages
t11: useChat returns { messages: [5 messages from store], sendMessage, ... }
t12: Component renders with 5 messages ✓

USER INTERACTION:
────────────────────────────────────────────────────────────────
t13: User types "Hello" and clicks send
t14: sendMessage({ text: "Hello" }) called
t15: AI SDK processes:
    - Creates user message (msg6)
    - chatHelpers.messages = [msg6]  // AI SDK only knows about new message
    - Sends to API
t16: Sync effect runs
    - Store has 5 messages, chatHelpers has 1
    - shouldSyncMessages = true (both have messages)
    - Syncs chatHelpers.messages to store
    - BUT WAIT! This would clear our history!

PROBLEM SCENARIO (if naive sync):
────────────────────────────────────────────────────────────────
    - Store would become: [msg6] (lost msg1-5!)

ACTUAL BEHAVIOR (AI SDK handles this):
────────────────────────────────────────────────────────────────
    - AI SDK's sendMessage preserves existing messages in its internal state
    - When transport sends request, it includes all messages for context
    - Response streaming updates messages, which then sync to store
    - Store ends up with: [msg1,msg2,msg3,msg4,msg5,msg6,msg7(assistant)]
```

### Key Insight: AI SDK's Internal Message Management

The AI SDK's `useChat` hook manages messages internally. When you call `sendMessage`, it:

1. **Preserves existing messages** in its internal state
2. **Appends the new user message**
3. **Sends all messages to the API** (for context)
4. **Receives and appends the assistant response**

The store sync ensures these updates propagate to the Zustand store, which then triggers React re-renders.

### Edge Cases and Solutions

#### Edge Case 1: Changing Chat Sessions

When navigating between chats, you need to reset the store:

```typescript
// Using chatKey to force re-mount
<Provider key={chatId} initialMessages={messages}>
  <ChatInterface />
</Provider>

// Or using the reset action
const { reset, setNewChat } = useChatActions();

const switchChat = async (newChatId: string) => {
  const newMessages = await loadChatHistory(newChatId);
  setNewChat(newChatId, newMessages);
};
```

#### Edge Case 2: Real-time Message Updates

If messages can be added externally (e.g., WebSocket), use store actions directly:

```typescript
const { pushMessage } = useChatActions();

useEffect(() => {
  const ws = new WebSocket('/realtime');
  ws.onmessage = (event) => {
    const newMessage = JSON.parse(event.data);
    pushMessage(newMessage);  // Directly updates store
  };
  return () => ws.close();
}, [pushMessage]);
```

#### Edge Case 3: Optimistic Updates

For instant UI feedback before server response:

```typescript
const { pushMessage, popMessage, sendMessage } = useChatActions();

const sendOptimistic = async (text: string) => {
  // 1. Add optimistic user message
  const optimisticMsg = {
    id: `temp-${Date.now()}`,
    role: 'user' as const,
    content: text,
    parts: [{ type: 'text' as const, text }],
  };
  pushMessage(optimisticMsg);

  try {
    // 2. Send actual message (will add real message + response)
    await sendMessage({ text });
    // 3. Remove optimistic message (real one from sendMessage takes over)
    popMessage();  // Remove the optimistic one
  } catch (error) {
    // 4. On error, remove optimistic and show error
    popMessage();
    throw error;
  }
};
```

### Performance Characteristics

| Scenario | Time Complexity | Notes |
|----------|-----------------|-------|
| Initial render with N messages | O(N) | Index building |
| sendMessage (user action) | O(1) | Append to array |
| Message lookup by ID | O(1) | Hash map lookup |
| Sync effect (per message update) | O(N) | Array copy + index rebuild |
| Throttled UI update | O(N) | Shallow copy of messages |

### Common Mistakes to Avoid

#### ❌ Passing initialMessages to useChat

```typescript
// WRONG - AI SDK's useChat doesn't use initialMessages the same way
const { messages } = useChat({
  initialMessages: serverMessages,  // This won't work as expected
});
```

#### ✅ Pass initialMessages to Provider

```typescript
// CORRECT - Provider initializes the store
<Provider initialMessages={serverMessages}>
  <Component />
</Provider>

// Then useChat reads from the store
const { messages } = useChat({ /* no initialMessages here */ });
```

#### ❌ Creating store on every render

```typescript
// WRONG - Creates new store every render, loses state
function BadProvider({ children, initialMessages }) {
  const store = createChatStore(initialMessages);  // New store each render!
  return <Provider store={store}>{children}</Provider>;
}
```

#### ✅ Use ref to persist store

```typescript
// CORRECT - Store persists across renders
function GoodProvider({ children, initialMessages }) {
  const storeRef = useRef(null);
  if (storeRef.current === null) {
    storeRef.current = createChatStore(initialMessages);
  }
  return <Provider store={storeRef.current}>{children}</Provider>;
}
```

---

## API Reference Summary

### Exports

```typescript
// Types
export type { UIMessage } from "@ai-sdk/react";
export type { ChatActions, StoreState, UseChatHelpers, UseChatOptions };
export type { DataPart, UseDataPartOptions, UseDataPartsReturn };

// Store Creation
export { createChatStore, createChatStoreCreator };

// Provider & Context
export { Provider, ChatStoreContext };

// Core Hooks
export { useChatStore, useChatStoreApi };
export { useChatMessages, useChatStatus, useChatError, useChatId };
export { useChatActions, useChatReset };

// Performance Hooks
export { useMessageById, useMessageCount, useMessageIds };
export { useVirtualMessages, useSelector };

// Enhanced useChat
export { useChat };

// Data Parts
export { useDataParts, useDataPart };

// Debug
export { debug, configureDebug, DebugLogger };
```
