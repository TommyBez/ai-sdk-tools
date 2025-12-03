# @ai-sdk-tools/agents

[![npm version](https://badge.fury.io/js/@ai-sdk-tools%2Fagents.svg)](https://badge.fury.io/js/@ai-sdk-tools%2Fagents)

Multi‑agent orchestration for the Vercel AI SDK v5. Coordinate specialists, automatic handoffs, typed context, memory, guardrails, and streaming telemetry with one class.

```bash
npm install @ai-sdk-tools/agents ai zod
# optional model providers
npm install @ai-sdk/openai @ai-sdk/anthropic @ai-sdk/google
```

## When to use it

- You want a router/orchestrator that can swap between specialists or escalate work mid-conversation.
- A workflow benefits from dedicated models (e.g. planning → execution → verification).
- You need server-side control over chat history, working memory, titles, and suggestions without re-sending the entire conversation from the client.
- LLM tools should be guarded, rate-limited, or cached and you want one place to configure it.

Use a single agent when the task is simple, latency is critical, or the tool graph is trivial.

## Feature highlights

- **Turn-key orchestration** – configurable rounds, tool ceilings, programmatic or LLM routing, and explicit `agentChoice`/`toolChoice`.
- **First-class memory** – load working memory, history, chat metadata, suggestions, and auto-title generation through `@ai-sdk-tools/memory`.
- **Typed execution context** – pass strongly-typed context into every tool through `createExecutionContext` and `getContext`.
- **Automatic handoffs** – attach specialists with `handoff()`; each handoff can filter conversation history or run side effects.
- **Guardrails & permissions** – run input/output guardrails and per-tool permissions before any tool is executed.
- **Streaming-friendly** – `toUIMessageStream` emits agent specific `data-*` parts (`agent-status`, `agent-handoff`, `rate-limit`) that plug directly into `@ai-sdk-tools/store`, devtools, or your UI.
- **Observability hooks** – structured `AgentEvent`s for every lifecycle change plus helper writers (`writeAgentStatus`, `writeRateLimit`, `writeDataPart`).

---

## Quick start – orchestrated support desk

```typescript
import { Agent, handoff } from '@ai-sdk-tools/agents';
import { openai } from '@ai-sdk/openai';

const technicalAgent = new Agent({
  name: 'Technical',
  model: openai('gpt-4o'),
  instructions: 'Handle product bugs and integrations.',
});

const billingAgent = new Agent({
  name: 'Billing',
  model: openai('gpt-4o-mini'),
  instructions: 'Answer pricing, invoices, refunds.',
});

const supportAgent = new Agent({
  name: 'Support Orchestrator',
  model: openai('gpt-4o-mini'),
  instructions: 'Own the thread. Route when a specialist is required.',
  matchOn: ['support', 'help'],
  handoffs: [
    technicalAgent,
    handoff(billingAgent, {
      inputFilter: (input) => ({
        ...input,
        inputHistory: input.inputHistory.slice(-8), // keep last 8 messages
      }),
      onHandoff: async (runContext) => {
        runContext.metadata = { ...runContext.metadata, reason: 'billing' };
      },
    }),
  ],
});

// app/api/chat/route.ts
export async function POST(req: Request) {
  const payload = await req.json();

  return supportAgent.toUIMessageStream({
    message: payload.message,         // last user message from the client
    context: {                        // anything the tools/memory need
      chatId: payload.chatId,
      userId: payload.userId,
      plan: payload.plan,
    },
    strategy: 'auto',                 // programmatic match before LLM routing
    maxRounds: 5,
    maxSteps: 10,
    onEvent(event) {
      if (event.type === 'agent-handoff') {
        console.log(`${event.from} → ${event.to}`, event.reason);
      }
    },
  });
}
```

Attach `@ai-sdk-tools/devtools` in development to see every data part in real time and use `@ai-sdk-tools/store/useDataPart('agent-status')` to drive UI indicators.

---

## Memory, history & chat automation

Working memory, conversation history, chat save/suggestions, and titles are configured per agent. Provide a `MemoryConfig` from `@ai-sdk-tools/memory` (in-memory, Redis, Upstash, Drizzle, etc.).

```typescript
import { InMemoryProvider } from '@ai-sdk-tools/memory/in-memory';

const agent = new Agent({
  name: 'Planner',
  model: openai('gpt-4o'),
  instructions: 'Plan initiatives with persistent context.',
  memory: {
    provider: new InMemoryProvider(),
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

When `workingMemory.enabled` is true every agent that actually performs work receives an `updateWorkingMemory` tool automatically. History + chat metadata are saved in `wrappedOnFinish`, so your frontend only sends the most recent user message.

---

## Typed execution context & tooling

```typescript
import {
  Agent,
  createExecutionContext,
  getContext,
} from '@ai-sdk-tools/agents';
import { tool } from 'ai';
import { z } from 'zod';

interface AppContext {
  userId: string;
  teamId: string;
  writer: never; // populated automatically
}

const fetchMetrics = tool({
  description: 'Fetch team metrics',
  parameters: z.object({ range: z.string() }),
  async execute(params, executionOptions) {
    const ctx = getContext<AppContext>(executionOptions);
    return db.metrics.find(ctx!.teamId, params.range);
  },
});

const agent = new Agent<AppContext>({
  name: 'Metrics Analyst',
  model: openai('gpt-4.1-mini'),
  instructions: ({ teamId }) =>
    `You are the analytics partner for team ${teamId}.`,
  tools: { fetchMetrics },
});

// Build a single typed context for every tool call
const execContext = createExecutionContext<AppContext>({
  writer: streamWriter,
  context: { userId, teamId },
});
agent.stream({ messages, executionContext: execContext });
```

---

## Routing, handoffs, and escalation

- **Programmatic routing** – Each specialist can expose `matchOn` (strings, regexes, or a predicate) so the orchestrator can choose instantly without an extra LLM call.
- **User hints** – `agentChoice` and `toolChoice` options in `toUIMessageStream` force the first round to start with the requested agent/tool.
- **LLM-driven handoffs** – If no programmatic match fires, the orchestrator uses the model and automatically yields a `handoff_to_agent` tool call. The default handoff tool copies the final system prompt and conversation history.
- **Custom filters** – Provide any `(input: HandoffInputData) => HandoffInputData` to redact files, trim the history window, or add summaries before the specialist sees the conversation.

```typescript
handoff(researchAgent, {
  inputFilter(input) {
    return {
      ...input,
      inputHistory: input.inputHistory.slice(-6),
      availableData: { ...input.availableData, summary: summarize(input) },
    };
  },
  async onHandoff(runContext) {
    await auditTrail.insert({
      from: runContext.metadata.agent,
      to: researchAgent.name,
    });
  },
});
```

If the LLM suggests a handoff, the orchestrator writes a transient `data-agent-handoff` part and emits `AgentEvent { type: 'agent-handoff' }`, making it easy to visualize in devtools or your own UI.

---

## Guardrails & permissions

```typescript
import {
  Agent,
  ToolPermissionDeniedError,
  runInputGuardrails,
  runOutputGuardrails,
} from '@ai-sdk-tools/agents';

const agent = new Agent({
  name: 'Regulated Agent',
  model: openai('gpt-4o'),
  instructions: 'Comply with regional policy.',
  inputGuardrails: [
    {
      name: 'pii-blocker',
      async execute({ input }) {
        if (hasPII(input)) return { tripwireTriggered: true };
        return { tripwireTriggered: false };
      },
    },
  ],
  outputGuardrails: [
    {
      name: 'citations-required',
      async execute({ agentOutput }) {
        return { tripwireTriggered: !agentOutput.includes('http') };
      },
    },
  ],
  permissions: {
    check: async ({ toolName, context }) => {
      const calls = context.usage.toolCalls[toolName] ?? 0;
      return { allowed: calls < 3 };
    },
  },
});
```

If a guardrail fails, `InputGuardrailTripwireTriggered` or `OutputGuardrailTripwireTriggered` is thrown. Permission failures raise `ToolPermissionDeniedError`.

---

## Streaming instrumentation

`toUIMessageStream` writes rich telemetry via helper writers:

- `writeAgentStatus` → `data-agent-status` parts for “routing / executing / completing”.
- `writeRateLimit` → `data-rate-limit` parts (limit, remaining, reset).
- `writeDataPart` → general purpose data channels that can be consumed with `@ai-sdk-tools/store/useDataPart`.

Pair this with `@ai-sdk-tools/devtools` to inspect SSE events, throttle noise, capture tool payloads, and view agent graphs. The same data parts power your UI so what you see in devtools is literally what the client receives.

---

## Integrations

- **`@ai-sdk-tools/memory`** – built‑in providers (in-memory, Redis, Upstash, Drizzle) plug into `memory.provider`.
- **`@ai-sdk-tools/store`** – drop-in replacement for `@ai-sdk/react` that understands agent data parts, artifacts, and streaming stores.
- **`@ai-sdk-tools/artifacts`** – stream typed components from any tool inside an agent; orchestration preserves those parts across handoffs.
- **`@ai-sdk-tools/cache`** – wrap expensive tools with `cached()` or `createCached()` to avoid repeated external calls even during streaming.
- **`@ai-sdk-tools/devtools`** – visualizes events, tool calls, agent flow, and state tabs. Works automatically because all agent streams emit structured events.

---

## API reference

### `class Agent<TContext>`

**Config**

| Option | Description |
| --- | --- |
| `name` | Unique agent identifier |
| `model` | Any AI SDK v5 language model |
| `instructions` | String or `(context) => string` system prompt |
| `tools` | Static object or function returning tools per context |
| `handoffs` | Array of `Agent` or `handoff(agent, config)` results |
| `maxTurns` | Maximum tool steps per round (default `10`) |
| `temperature`, `modelSettings` | Forwarded to the AI SDK agent |
| `matchOn` | Programmatic routing (string, regex, or predicate) |
| `onEvent` | Receives every `AgentEvent` |
| `inputGuardrails`, `outputGuardrails` | Validation pipelines |
| `permissions` | Tool permission checker |
| `memory` | `MemoryConfig` from `@ai-sdk-tools/memory` |
| `lastMessages` | Override context window per agent (default 10 for orchestrators, 5 for specialists) |

**Methods**

- `generate(options)` – synchronous convenience wrapper.
- `stream(options)` – returns the underlying AI SDK stream.
- `toUIMessageStream(options)` – Next.js compatible `Response`.
- `getHandoffs()` and `getConfiguredHandoffs()` – inspect downstream agents.

### Context helpers

- `AgentRunContext` – a mutable object that travels across handoffs. Add metadata or store run-level info.
- `createExecutionContext({ context, writer, metadata })` – build the `experimental_context` passed into every tool. Automatically injects the stream writer and metadata.
- `getContext(executionOptions)` – typed accessor inside a tool to recover whatever you placed in the execution context.
- `ContextOptions` and `ExecutionContext` types.

### Handoff utilities

- `handoff(agent, config?)` – wrap an agent with optional `inputFilter` and `onHandoff`.
- `createHandoff(targetAgent, context?, reason?)` – build manual instructions (useful if you need to queue a handoff yourself).
- `createHandoffTool(agents)` – attach the standard `handoff_to_agent` tool to any agent.
- `isHandoffTool`, `isHandoffResult`, `getTransferMessage`.

### Guardrails & permissions

- Base errors: `AgentsError`, `InputGuardrailTripwireTriggered`, `OutputGuardrailTripwireTriggered`, `GuardrailExecutionError`, `MaxTurnsExceededError`, `ToolCallError`, `ToolPermissionDeniedError`.
- Helpers: `runInputGuardrails`, `runOutputGuardrails`, `checkToolPermission`, `createUsageTracker`, `trackToolCall`.

### Routing utilities

- `matchAgent(agent, message, matchOn?)` – score an individual agent.
- `findBestMatch(message, agents, selector?)` – return the top match for programmatic routing.

### Streaming writers

- `writeAgentStatus`, `writeDataPart`, `writeRateLimit` – convenience writers for UI message streams.

### Types

`AgentConfig`, `AgentEvent`, `AgentGenerateOptions`, `AgentGenerateResult`, `AgentStreamOptions`, `AgentStreamOptionsUI`, `AgentStreamResult`, `AgentUIMessage`, `AgentDataParts`, `ConfiguredHandoff`, `HandoffConfig`, `HandoffData`, `MemoryIdentifiers`, `GuardrailResult`, `InputGuardrail`, `OutputGuardrail`, `ToolPermissions`, `ToolPermissionContext`, `ToolPermissionResult`, plus every class/utility listed above are exported.

---

## Requirements

- Node.js 18+
- Vercel AI SDK v5.0.0 or later
- Any ESM-compatible runtime (Next.js / Vite / Bun / etc.)

## Resources

- Examples: `apps/example/src/ai/agents`
- Devtools UI: `packages/devtools`
- Memory providers & schema helpers: `packages/memory`

## License

MIT © [Midday](https://midday.ai)
