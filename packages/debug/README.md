# @ai-sdk-tools/debug

Zero-dependency logging helpers for the AI SDK Tools ecosystem. Instead of bundling Pino or Winston everywhere we ship a tiny wrapper that prints colorized logs only when you explicitly opt in.

---

## Installation

```bash
npm install @ai-sdk-tools/debug
```

---

## Usage

```ts
import { createLogger, logger } from '@ai-sdk-tools/debug';

const agentLogger = createLogger('AGENT');

agentLogger.debug('Starting stream', { name: 'triage' });
agentLogger.info('Handoff detected', { from: 'Triage', to: 'Billing' });
agentLogger.warn('Retrying tool call', { tool: 'weather' });
agentLogger.error('Stream failed', { error });

// Need a quick global log?
logger.debug('Memory syncing…');
```

- Each logger is scoped by category (`[AGENT]`, `[MEMORY]`, …) so you can filter easily.
- When disabled the factory returns no-op functions, so there is effectively **zero overhead**.

---

## Enable logging

Set `DEBUG_AGENTS=true` in whichever process you want to inspect:

```bash
DEBUG_AGENTS=true npm run dev
```

Without that variable the package does not write to the console.

---

## Output format

```
[12:32:04.215] INFO [AGENT] Handoff complete {"from":"Triage","to":"Specialist"}
```

- Timestamp (HH:MM:SS.mmm)
- Level (`DEBUG`, `INFO`, `WARN`, `ERROR`) with ANSI colors
- Category (`[AGENT]`, `[MEMORY]`, etc.)
- Message + optional JSON payload

Works in Node.js, browsers, Edge, Bun — anywhere `console` exists.

---

## API

### `createLogger(category: string)`

Returns `{ debug, info, warn, error }`, each accepting `(message: string, data?: unknown)`.

### `logger`

A pre-built unscoped logger (`[DEBUG]`) that honors the same environment variable.

---

## License

MIT © [Midday](https://midday.ai)
