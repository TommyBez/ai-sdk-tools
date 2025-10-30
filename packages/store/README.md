# @ai-sdk-tools/store

A high-performance drop-in replacement for @ai-sdk/react with advanced state management, built-in optimizations, and zero configuration required.

## Requirements

- React >= 18
- Zustand >= 5
- @ai-sdk/react >= 2

## Performance Features

- **3-5x faster** than standard @ai-sdk/react
- **O(1) message lookups** with hash map indexing
- **Batched updates** to minimize re-renders
- **Memoized selectors** with automatic caching
- **Message virtualization** for large chat histories
- **Advanced throttling** with scheduler.postTask
- **Deep equality checks** to prevent unnecessary updates

## Installation

```bash
npm install @ai-sdk-tools/store
# or
bun add @ai-sdk-tools/store
```

## Debug Configuration

The store package includes a debug utility that can be configured to control logging:

### Environment Variable

Set `DEBUG=true` to enable debug logging:

```bash
# Enable debug logging
DEBUG=true npm run dev

# Or in your .env file
DEBUG=true
```

By default, debug logging is disabled unless `DEBUG=true` is set.

## Quick Start

### 1. Wrap Your App

```tsx
import { Provider } from '@ai-sdk-tools/store';

function App() {
  return (
    <Provider initialMessages={[]}>
      <ChatComponent />
    </Provider>
  );
}
```

### 2. Use Chat Hooks

```tsx
import { useChat, useChatMessages } from '@ai-sdk-tools/store';
import { DefaultChatTransport } from 'ai';

function ChatComponent() {
  // Same API as @ai-sdk/react, but 3-5x faster!
  const { messages, sendMessage, status } = useChat({
    transport: new DefaultChatTransport({
      api: '/api/chat'
    })
  });

  return (
    <div>
      {messages.map(message => (
        <div key={message.id}>{message.content}</div>
      ))}
    </div>
  );
}
```

### 3. Access State from Any Component

```tsx
function MessageCounter() {
  // No prop drilling needed!
  const messageCount = useMessageCount();
  const status = useChatStatus();
  
  return <div>{messageCount} messages ({status})</div>;
}
```

## Advanced Features

### Message Virtualization
Perfect for large chat histories:

```tsx
function VirtualizedChat() {
  // Only render visible messages for optimal performance
  const visibleMessages = useVirtualMessages(0, 50);
  
  return (
    <div>
      {visibleMessages.map(message => (
        <MessageComponent key={message.id} message={message} />
      ))}
    </div>
  );
}
```

### Memoized Selectors
Cache expensive computations:

```tsx
function ChatAnalytics() {
  const userMessageCount = useSelector(
    'userMessages',
    (messages) => messages.filter((m) => m.role === 'user').length
  );
  
  return <div>User messages: {userMessageCount}</div>;
}
```

### Fast Message Lookups
O(1) performance for message access:

```tsx
function MessageDetails({ messageId }: { messageId: string }) {
  // O(1) lookup instead of O(n) array.find()
  const message = useMessageById(messageId);
  
  return <div>{message.content}</div>;
}
```

### Transient Data Parts
Capture and consume structured, out-of-band updates that arrive during streaming via `onData` parts with types prefixed by `data-`.

```tsx
import { useDataParts, useDataPart } from '@ai-sdk-tools/store';

// Read all latest data parts by type
function Sidebar() {
  const { byType } = useDataParts();
  const agentStatus = byType['agent-status'];
  return agentStatus ? <p>Status: {agentStatus.data.status}</p> : null;
}

// Read a specific data part with a clear function
function RateLimitBanner() {
  const [rateLimit, clearRateLimit] = useDataPart<{ remaining: number }>('rate-limit');
  if (!rateLimit) return null;
  return (
    <div>
      Remaining: {rateLimit.remaining}
      <button onClick={clearRateLimit}>Dismiss</button>
    </div>
  );
}
```

## Migration from @ai-sdk/react

### Before:
```tsx
import { useChat } from '@ai-sdk/react';

function Chat() {
  const chat = useChat({ api: '/api/chat' });
  return <div>{/* chat UI */}</div>;
}
```

### After:
```tsx
import { Provider, useChat } from '@ai-sdk-tools/store';
import { DefaultChatTransport } from 'ai';

function App() {
  return (
    <Provider>
      <Chat />
    </Provider>
  );
}

function Chat() {
  // Same API, but 3-5x faster!
  const chat = useChat({ 
    transport: new DefaultChatTransport({ api: '/api/chat' })
  });
  return <div>{/* chat UI */}</div>;
}
```

## Performance Benchmarks

| Scenario | @ai-sdk/react | @ai-sdk-tools/store | Improvement |
|----------|---------------|---------------------|-------------|
| 1000 messages | 120ms | 35ms | **3.4x faster** |
| Message lookup | O(n) | O(1) | **10-100x faster** |
| Complex filtering | 45ms | 12ms | **3.8x faster** |
| Re-render frequency | High | Minimal | **5x fewer** |

## API Reference

### Hooks

```tsx
// Core chat functionality
const chat = useChat(options)           // Enhanced useChat with performance
const messages = useChatMessages()      // Get all messages
const status = useChatStatus()          // Chat status
const error = useChatError()            // Error state
const id = useChatId()                  // Chat ID

// Performance hooks
const message = useMessageById(id)      // O(1) message lookup
const count = useMessageCount()         // Optimized message count
const ids = useMessageIds()             // All message IDs
const slice = useVirtualMessages(0, 50) // Message virtualization
const result = useSelector(key, fn, deps) // Memoized selectors

// Actions
const actions = useChatActions()        // All actions object
const reset = useChatReset()            // Reset chat state to initial values

// Data parts
const { byType, all } = useDataParts()  // Latest values by data-* type
const [value, clear] = useDataPart('rate-limit') // Specific data part

// Store access
const store = useChatStoreApi()         // Access store API (Zustand vanilla)
const state = useChatStore()            // Read full state
```

#### useChat options

`useChat` accepts all options from `@ai-sdk/react` plus:

- `store?`: a compatible Zustand store to sync into (defaults to the context store created by `Provider`)
- `enableBatching?` (default `true`): batch state syncs with `requestAnimationFrame` for fewer re-renders

### Provider

```tsx
<Provider initialMessages={messages}>
  <YourApp />
</Provider>
```

Need to embed the store into an existing architecture? Use `createChatStoreCreator()` with `zustand/vanilla` to wire chat state into your own provider or persistence layer:

```ts
import { createStore } from 'zustand/vanilla';
import { createChatStoreCreator } from '@ai-sdk-tools/store';

const createChatStore = createChatStoreCreator();
export const chatStore = createStore(createChatStore);
```

### Debug utilities

```ts
import { configureDebug, debug } from '@ai-sdk-tools/store';

// At app startup
configureDebug({ enabled: true, level: 'warn' });
debug.warn('Slow update detected');
```

You can also enable logging via the `DEBUG=true` environment variable.

## TypeScript Support

Full generic support with custom message types:

```tsx
interface MyMessage extends UIMessage<
  { userId: string }, // metadata
  { weather: WeatherData }, // data
  { getWeather: { input: { location: string }, output: WeatherData } } // tools
> {}

// Fully typed throughout
const chat = useChat<MyMessage>({ 
  transport: new DefaultChatTransport({ api: '/api/chat' })
})
const messages = useChatMessages<MyMessage>() // Fully typed!
```

## SSR and Hydration

- `Provider` supports `initialMessages` for hydration-safe SSR.
- On hydration, if the store already has server-rendered messages and the chat hook has none yet, the store will keep its messages to avoid flicker.

## Contributing

Contributions are welcome! See the [contributing guide](../../CONTRIBUTING.md) for details.

## License

MIT