# ai-sdk-tools

Complete toolkit for building advanced AI applications with the [Vercel AI SDK](https://sdk.vercel.ai/). This package provides everything you need: multi-agent orchestration, state management, caching, artifact streaming, development tools, and persistent memory.

## Installation

```bash
npm install ai-sdk-tools
```

This installs all tools in a single package with namespaced exports.

### Peer Dependencies

Depending on which features you use, you may need to install:

```bash
npm install ai @ai-sdk/react react react-dom zod zustand
```

## What's Included

This package includes all AI SDK tools:

- **`agents`** - Multi-agent orchestration with handoffs and routing
- **`artifacts`** - Structured artifact streaming for React components  
- **`cache`** - Universal caching for AI tool executions
- **`devtools`** - Development and debugging tools
- **`memory`** - Persistent memory system for AI agents
- **`store`** - Zustand-based state management for AI applications

## Usage

Import tools directly from the package:

```typescript
import { Agent, artifact, cached, useChat, AIDevtools } from 'ai-sdk-tools';
```

### Multi-Agent Orchestration

Build intelligent workflows with specialized agents:

```typescript
import { Agent, handoff } from 'ai-sdk-tools';
import { openai } from '@ai-sdk/openai';

const supportAgent = new Agent({
  model: openai('gpt-4o'),
  name: 'SupportAgent',
  instructions: 'You handle customer support queries.',
});

const billingAgent = new Agent({
  model: openai('gpt-4o'),
  name: 'BillingAgent',
  instructions: 'You handle billing and payment issues.',
});

const router = new Agent({
  model: openai('gpt-4o-mini'),
  name: 'Router',
  instructions: 'Route each message to the best specialist.',
  handoffs: [handoff(supportAgent), handoff(billingAgent)],
});

const result = await router.generate({
  prompt: 'I need help with my invoice',
});

console.log(result.finalAgent); // => 'BillingAgent'
```

### State Management

Manage chat state globally with Zustand:

```typescript
import { useChat } from 'ai-sdk-tools';
import { DefaultChatTransport } from 'ai';

export const useChatHook = useChat({
  transport: new DefaultChatTransport({ api: '/api/chat' }),
});

function ChatComponent() {
  const { messages, input, handleInputChange, handleSubmit } = useChatHook();
  return (
    <form onSubmit={handleSubmit}>
      {messages.map((msg) => (
        <div key={msg.id}>{msg.content}</div>
      ))}
      <input value={input} onChange={handleInputChange} />
    </form>
  );
}
```

### Tool Caching

Cache expensive tool executions to reduce costs and improve performance:

```typescript
import { cached } from 'ai-sdk-tools';
import { tool } from 'ai';
import { z } from 'zod';

const weatherTool = cached(
  tool({
    description: 'Get weather for a location',
    inputSchema: z.object({
      location: z.string(),
    }),
    execute: async ({ location }) => {
      const response = await fetch(`https://api.weather.com/${location}`);
      return response.json();
    },
  }),
  { ttl: 3600 }
);
```

### Artifact Streaming

Stream structured artifacts from AI tools to React components:

```typescript
import { artifact, getWriter } from 'ai-sdk-tools';
import { tool } from 'ai';
import { z } from 'zod';

const chartArtifact = artifact('chart', z.object({
  status: z.enum(['loading', 'complete']).default('loading'),
  points: z.array(z.number()).default([]),
}));

export const generateChart = tool({
  description: 'Build a chart artifact',
  inputSchema: z.object({ values: z.array(z.number()) }),
  async execute({ values }, executionOptions) {
    const writer = getWriter(executionOptions);
    const stream = chartArtifact.stream({ points: [] }, writer);

    await stream.update({ status: 'loading' });
    await stream.complete({ status: 'complete', points: values });

    return 'Chart ready';
  },
});

// In React components import hooks from '@ai-sdk-tools/artifacts/client'
// const { data, status } = useArtifact(chartArtifact);
```

### Development Tools

Debug and monitor your AI applications in real-time:

```typescript
import { AIDevtools } from 'ai-sdk-tools';

function App() {
  return (
    <>
      <YourChatComponent />
      <AIDevtools />
    </>
  );
}
```

### Persistent Memory

Add long-term memory to your agents:

```typescript
import { Agent } from 'ai-sdk-tools';
import { openai } from '@ai-sdk/openai';
import { InMemoryProvider } from '@ai-sdk-tools/memory/providers/in-memory';

const memoryProvider = new InMemoryProvider();

const agent = new Agent({
  model: openai('gpt-4o'),
  name: 'AssistantAgent',
  instructions: 'You are a helpful assistant with memory.',
  memory: {
    provider: memoryProvider,
    workingMemory: { enabled: true, scope: 'chat' },
    history: { enabled: true, limit: 10 },
  },
});

// Agent can now remember context across conversations
```

### Streaming Data Parts

Send transient status updates and rate limit information alongside chat responses:

```typescript
import type { UIMessageStreamWriter } from 'ai';
import {
  Agent,
  writeAgentStatus,
  writeDataPart,
  writeRateLimit,
  writeSuggestions,
} from 'ai-sdk-tools';

let writerRef: UIMessageStreamWriter | null = null;

const assistant = new Agent({ /* ... */ });

return assistant.toUIMessageStream({
  message,
  context,
  beforeStream: ({ writer }) => {
    writerRef = writer;
    writeRateLimit(writer, { limit: 100, remaining: 97, reset: new Date().toISOString() });
    return true;
  },
  onEvent: (event) => {
    if (!writerRef) return;
    if (event.type === 'agent-start') {
      writeAgentStatus(writerRef, { status: 'executing', agent: event.agent });
    }
    if (event.type === 'agent-handoff') {
      writeDataPart(writerRef, 'data-agent-handoff', {
        from: event.from,
        to: event.to,
        reason: event.reason,
      });
    }
    if (event.type === 'agent-complete') {
      writeSuggestions(writerRef, ['Summarize', 'Create action items']);
    }
  },
});
```

On the client you can consume these via `@ai-sdk-tools/store` hooks such as `useDataPart('rate-limit')` or `useDataParts()`.

## Individual Packages

If you only need specific tools, you can install them individually:

```bash
npm install @ai-sdk-tools/agents
npm install @ai-sdk-tools/artifacts
npm install @ai-sdk-tools/cache
npm install @ai-sdk-tools/devtools
npm install @ai-sdk-tools/memory
npm install @ai-sdk-tools/store
```

Each package can be used independently with its own API. See individual package documentation for details.

## Documentation

- [Full Documentation](https://aisdk.tools)
- [API Reference](https://aisdk.tools/docs)
- [Examples](https://github.com/midday-ai/ai-sdk-tools/tree/main/apps/example)

## Features

✅ **Multi-agent orchestration** - Coordinate multiple specialized agents  
✅ **State management** - Global state for AI applications  
✅ **Universal caching** - Cache any tool execution  
✅ **Artifact streaming** - Structured real-time updates  
✅ **Development tools** - Debug and monitor AI apps  
✅ **Persistent memory** - Long-term agent memory  
✅ **TypeScript first** - Full type safety throughout  
✅ **Provider agnostic** - Works with any AI SDK provider  
✅ **Production ready** - Battle-tested in real applications  

## Requirements

- Node.js 18+
- AI SDK v5.0.0 or higher
- React 18+ (for React-specific features)

## License

MIT © [Midday](https://midday.ai)

## Links

- [GitHub](https://github.com/midday-ai/ai-sdk-tools)
- [Issues](https://github.com/midday-ai/ai-sdk-tools/issues)
- [Discussions](https://github.com/midday-ai/ai-sdk-tools/discussions)

