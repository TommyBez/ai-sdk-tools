# @ai-sdk-tools/debug

Lightweight debug logger for AI SDK Tools packages with zero-dependency, colorized console output.

## Installation

```bash
npm install @ai-sdk-tools/debug
```

## Usage

```typescript
import { createLogger } from '@ai-sdk-tools/debug';

const logger = createLogger('AGENT');

// Log at different levels
logger.debug("Starting stream", { name: "reports" });
logger.info("Handoff detected", { targetAgent: "operations" });
logger.warn("No text accumulated during streaming");
logger.error("Failed to load", { error });
```

## Environment Variables

- `DEBUG_AGENTS=true` - Enable debug logging with colorized output
- No `DEBUG_AGENTS` or `DEBUG_AGENTS=false` - Silent mode (no logging)

## Features

- **Zero dependencies**: Uses native `console` APIs, no transports or bindings required
- **Zero overhead when disabled**: Returns no-op functions unless `DEBUG_AGENTS=true`
- **Colorized output**: Timestamped, ANSI-colored log lines for quick scanning
- **Category-based logging**: Scope logs per package or feature (e.g., `AGENT`, `MEMORY`, `CACHE`)
- **Runtime agnostic**: Safe to use in Node.js, Next.js route handlers, and edge environments
- **Drop-in usage**: Same API whether debug mode is enabled or not

## API

### `createLogger(category: string)`

Creates a category-scoped logger.

```typescript
const logger = createLogger('MY_CATEGORY');

logger.debug(message: string, data?: any);
logger.info(message: string, data?: any);
logger.warn(message: string, data?: any);
logger.error(message: string, data?: any);
```

## License

MIT

