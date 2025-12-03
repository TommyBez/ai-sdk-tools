# @ai-sdk-tools/memory

Persistent working memory, conversation history, and chat metadata for agents built with the AI SDK. Pick a built-in provider (in-memory, Redis, Upstash, Drizzle) or implement the tiny `MemoryProvider` interface to plug in your own store.

---

## Installation

```bash
npm install @ai-sdk-tools/memory
```

Optional peer deps (install the ones you need):

```bash
# SQL / Drizzle support
npm install drizzle-orm

# Redis clients
npm install redis           # or
npm install ioredis

# Upstash (edge/serverless)
npm install @upstash/redis
```

Providers are exposed via subpath exports:

```ts
import { InMemoryProvider } from '@ai-sdk-tools/memory/in-memory';
import { RedisProvider } from '@ai-sdk-tools/memory/redis';
import { UpstashProvider } from '@ai-sdk-tools/memory/upstash';
import { DrizzleProvider } from '@ai-sdk-tools/memory/drizzle';
```

---

## Why use it?

- **Working memory** – persist user preferences/facts beyond the context window and give agents an `updateWorkingMemory` tool with your own template.
- **Server-side history** – the frontend only sends the newest message while the agent loads the last _n_ messages from storage.
- **Chat metadata** – auto-generate chat titles and prompt suggestions, list chats per user, delete sessions, etc.

Everything is expressed through a single `MemoryConfig` you pass to an `Agent`.

---

## Quick start with `Agent`

```ts
import { Agent } from '@ai-sdk-tools/agents';
import { openai } from '@ai-sdk/openai';
import { InMemoryProvider } from '@ai-sdk-tools/memory/in-memory';

const memory = new InMemoryProvider();

const agent = new Agent({
  name: 'Planner',
  model: openai('gpt-4o'),
  instructions: 'Help plan projects. Update working memory when the user shares preferences.',
  memory: {
    provider: memory,
    workingMemory: {
      enabled: true,
      scope: 'user',
      template: `# Working Memory\n\n## Preferences\n- ...`,
    },
    history: {
      enabled: true,
      limit: 12,
    },
    chats: {
      enabled: true,
      generateTitle: true,
      generateSuggestions: {
        enabled: true,
        limit: 4,
      },
    },
  },
});
```

In your streaming endpoint pass context with `chatId` / `userId`:

```ts
return agent.toUIMessageStream({
  message,
  context: { chatId, userId },
});
```

The agent will:
1. Load working memory + history before the first token.
2. Inject `updateWorkingMemory` if `workingMemory.enabled`.
3. Save user/assistant messages via `memory.provider.saveMessage`.
4. Manage chat metadata (titles/suggestions) when `chats.enabled`.

---

## Providers

### InMemory (development)

```ts
import { InMemoryProvider } from '@ai-sdk-tools/memory/in-memory';
const memory = new InMemoryProvider();
```

Great for local development and integration tests. Everything stays in process memory.

### Redis (self-hosted / managed)

```ts
import { createClient } from 'redis';
import { RedisProvider } from '@ai-sdk-tools/memory/redis';

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

const memory = new RedisProvider(redis, {
  prefix: 'my-app:memory:',
  messageTtl: 60 * 60 * 24 * 30, // optional
});
```

Works with `redis`, `ioredis`, or any client exposing the same commands. Perfect for Node services or background workers.

### Upstash (serverless / edge)

```ts
import { Redis } from '@upstash/redis';
import { UpstashProvider } from '@ai-sdk-tools/memory/upstash';

const memory = new UpstashProvider(Redis.fromEnv(), {
  prefix: 'my-app:memory:',
});
```

Uses the HTTP API so it runs anywhere (Next.js Edge, Cloudflare Workers, etc.).

### Drizzle (SQL databases)

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import { pgTable, text, timestamp } from 'drizzle-orm/pg-core';
import { DrizzleProvider } from '@ai-sdk-tools/memory/drizzle';

const workingMemoryTable = pgTable('working_memory', {
  id: text('id').primaryKey(),
  scope: text('scope').notNull(),
  chatId: text('chat_id'),
  userId: text('user_id'),
  content: text('content').notNull(),
  updatedAt: timestamp('updated_at').notNull(),
});

const messagesTable = pgTable('conversation_messages', {
  id: text('id').primaryKey(),
  chatId: text('chat_id').notNull(),
  userId: text('user_id'),
  role: text('role').notNull(),
  content: text('content').notNull(),
  timestamp: timestamp('timestamp').notNull(),
});

const memory = new DrizzleProvider(db, {
  workingMemoryTable,
  messagesTable,
  chatsTable, // optional: only if you need chat lists/titles
});
```

See [`DRIZZLE.md`](./DRIZZLE.md) for schema helpers, migrations, and examples (Postgres, MySQL, SQLite/Turso).

---

## Memory configuration reference

```ts
interface MemoryConfig {
  provider: MemoryProvider;
  workingMemory?: {
    enabled: boolean;
    scope: 'chat' | 'user';
    template?: string;
  };
  history?: {
    enabled: boolean;
    limit?: number;
  };
  chats?: {
    enabled: boolean;
    generateTitle?: boolean | GenerateTitleConfig;
    generateSuggestions?: boolean | GenerateSuggestionsConfig;
  };
}
```

- **Working memory scope**  
  - `chat`: per-conversation (recommended for most assistants).  
  - `user`: shared across all chats for the same user (preferences/traits).
- **History.limit** – how many messages to load from storage before streaming starts (default `undefined`, agent falls back to `lastMessages`).
- **Chats.generateTitle** – customize the model/instructions used to name chats.
- **Chats.generateSuggestions** – produce post-response suggestions using structured output (`generateObject` under the hood).

Utility exports:

- `DEFAULT_TEMPLATE`
- `formatWorkingMemory(memory)`
- `formatHistory(messages, limit?)`

Use them if you build your own prompts.

---

## MemoryProvider interface

Implement two required methods (plus optional ones if you need history/chats):

```ts
import type {
  MemoryProvider,
  MemoryScope,
  WorkingMemory,
  ConversationMessage,
  ChatSession,
} from '@ai-sdk-tools/memory';

class CustomProvider implements MemoryProvider {
  async getWorkingMemory({ chatId, userId, scope }: {
    chatId?: string;
    userId?: string;
    scope: MemoryScope;
  }): Promise<WorkingMemory | null> {
    // Fetch from your store
  }

  async updateWorkingMemory(params: {
    chatId?: string;
    userId?: string;
    scope: MemoryScope;
    content: string;
  }): Promise<void> {
    // Persist Markdown content + timestamp
  }

  async saveMessage?(message: ConversationMessage): Promise<void> {}
  async getMessages?<T = any>(params: { chatId: string; userId?: string; limit?: number }): Promise<T[]> {}
  async saveChat?(chat: ChatSession): Promise<void> {}
  async getChats?(params: { userId?: string; search?: string; limit?: number }): Promise<ChatSession[]> {}
  async getChat?(chatId: string): Promise<ChatSession | null> {}
  async updateChatTitle?(chatId: string, title: string): Promise<void> {}
  async deleteChat?(chatId: string): Promise<void> {}
}
```

`ConversationMessage.content` can be a string or serialized AI SDK message; providers typically store JSON and parse it before handing it back.

---

## Types

- `WorkingMemory { content: string; updatedAt: Date }`
- `ConversationMessage { chatId; userId?; role; content; timestamp }`
- `ChatSession { chatId; userId?; title?; createdAt; updatedAt; messageCount }`
- `ChatsConfig`, `GenerateTitleConfig`, `GenerateSuggestionsConfig`
- `MemoryScope = 'chat' | 'user'`
- `MemoryConfig`, `MemoryProvider`

All of these ship with TypeScript definitions so you can build providers confidently.

---

## Tips

- **Context is critical** – your app must supply `chatId` (and optionally `userId`) in the agent context so the provider can segment data correctly.
- **Combining scopes** – use `scope: 'user'` for preferences + `history.limit` to keep per-chat transcripts short.
- **TTL strategy** – for Redis/Upstash providers, use `messageTtl` to automatically prune old conversations.
- **Chat lists** – implement `getChats`/`saveChat` if you want “recent chats” UI without writing additional services.

---

## License

MIT © [Midday](https://midday.ai)
