# Memoria Development Guide

> Spec-driven development approach for OpenClaw Memoria, based on codebase research and established project patterns.

---

## Overview

This document outlines the recommended development approach for implementing the Memoria memory system, following OpenClaw's existing patterns for testing, plugin architecture, and code organization.

---

## 1. Interface-First Design (Contracts)

Start with type definitions before any implementation. Create `src/memoria/types.ts`:

```typescript
// =============================================================================
// Core Event Types
// =============================================================================

export type MemoriaEventType =
  // Compaction lifecycle (CRITICAL for data loss prevention)
  | 'compaction:warning'    // 80% context used
  | 'compaction:imminent'   // 95% context used
  | 'compaction:pre'        // BEFORE any pruning
  | 'compaction:executing'  // During compaction
  | 'compaction:post'       // After compaction
  | 'compaction:failed'     // Compaction error

  // Message lifecycle
  | 'message:received'
  | 'message:sending'
  | 'message:sent'

  // Agent lifecycle
  | 'agent:start'
  | 'agent:end'
  | 'session:start'
  | 'session:end'

  // Tool lifecycle
  | 'tool:before'
  | 'tool:after'

  // Memory operations
  | 'memory:write'
  | 'memory:search'
  | 'memory:flush';

export type MemoriaEvent = {
  id: string;
  timestamp: number;
  agentId: string;
  sessionKey: string;
  type: MemoriaEventType;
  payload: unknown;
};

// =============================================================================
// Memory Operations
// =============================================================================

export type RetainOptions = {
  category?: string;
  confidence?: number;
  metadata?: Record<string, unknown>;
};

export type RecallOptions = {
  limit?: number;
  minScore?: number;
  backends?: string[];
  sessionKey?: string;
};

export type MemoriaResult = {
  content: string;
  score: number;
  source: string;
  metadata?: Record<string, unknown>;
};

export type Insight = {
  topic: string;
  content: string;
  confidence: number;
  sources: string[];
};

// =============================================================================
// Unified Client Interface
// =============================================================================

export interface MemoriaClient {
  // Core operations
  retain(content: string, options?: RetainOptions): Promise<void>;
  recall(query: string, options?: RecallOptions): Promise<MemoriaResult[]>;
  reflect(topic?: string): Promise<Insight[]>;

  // Event operations
  subscribe(event: MemoriaEventType, handler: EventHandler): () => void;
  emit(event: MemoriaEventType, payload: unknown): Promise<void>;

  // Multi-backend search
  search(query: string, options?: SearchOptions): Promise<UnifiedResult[]>;

  // Lifecycle
  close(): Promise<void>;
}

// =============================================================================
// Backend Plugin Interface
// =============================================================================

export interface MemoriaBackend {
  id: string;
  name: string;

  // Core operations
  onRetain(content: string, meta: RetainOptions): Promise<void>;
  onRecall(query: string, opts: RecallOptions): Promise<MemoriaResult[]>;

  // Optional capabilities
  onReflect?(topic: string): Promise<Insight[]>;
  onTemporalQuery?(query: string, asOf: Date): Promise<MemoriaResult[]>;

  // Compaction hooks (CRITICAL)
  onCompactionPre?(event: CompactionPreEvent): Promise<void>;
  onCompactionPost?(event: CompactionPostEvent): Promise<void>;

  // Lifecycle
  start?(): Promise<void>;
  stop?(): Promise<void>;
  healthCheck?(): Promise<{ ok: boolean; error?: string }>;
}

// =============================================================================
// Compaction Events
// =============================================================================

export type CompactionPreEvent = {
  sessionId: string;
  messages: unknown[];
  tools: unknown[];
  context: unknown;
  tokenCount: number;
  timestamp: number;
};

export type CompactionPostEvent = {
  summary: string;
  tokensBefore: number;
  tokensAfter: number;
  messagesRemoved: number;
};
```

---

## 2. Test Harness Structure

Follow the project's 3-tier testing approach:

### Directory Structure

```
src/memoria/
├── types.ts                    # Contracts (interface-first)
├── event-bus.ts                # Event emission layer
├── event-bus.test.ts           # Unit tests
├── client.ts                   # Unified MemoriaClient
├── client.test.ts              # Contract compliance tests
├── backends/
│   ├── types.ts                # Backend interface
│   ├── hindsight.ts            # Hindsight integration
│   ├── hindsight.test.ts       # Mocked HTTP tests
│   ├── hindsight.live.test.ts  # Real API tests (gated)
│   ├── graphiti.ts             # Graphiti integration
│   ├── graphiti.test.ts
│   ├── nats.ts                 # NATS JetStream
│   ├── nats.test.ts
│   └── local.ts                # Always-available fallback
├── compaction/
│   ├── events.ts               # Pre/post compaction events
│   ├── events.test.ts          # Timing verification
│   ├── protection.ts           # Data loss prevention
│   └── protection.test.ts
└── integration.e2e.test.ts     # Full flow with stubs

extensions/
├── memoria-hindsight/
│   ├── package.json
│   ├── openclaw.plugin.json
│   ├── index.ts
│   └── index.test.ts
├── memoria-graphiti/
│   └── ...
└── memoria-nats/
    └── ...
```

### Test Configuration

Add to existing Vitest config patterns:

```typescript
// vitest.config.ts additions
{
  include: [
    'src/**/*.test.ts',
    'extensions/**/*.test.ts',
  ],
  exclude: [
    '**/*.live.test.ts',
    '**/*.e2e.test.ts',
  ],
}
```

---

## 3. Test Patterns

### A. Event Sequence Verification

Based on `src/infra/agent-events.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { createMemoriaEventBus } from './event-bus.js';

describe('Memoria Event Bus', () => {
  let bus: MemoriaEventBus;
  let events: MemoriaEvent[];

  beforeEach(() => {
    events = [];
    bus = createMemoriaEventBus();
  });

  it('maintains event ordering', async () => {
    bus.subscribe('compaction:pre', (e) => events.push(e));
    bus.subscribe('compaction:post', (e) => events.push(e));

    await bus.emit('compaction:pre', { tokenCount: 50000 });
    await bus.emit('compaction:post', { tokensAfter: 10000 });

    expect(events.map(e => e.type)).toEqual(['compaction:pre', 'compaction:post']);
  });

  it('returns unsubscribe function', () => {
    const handler = vi.fn();
    const unsub = bus.subscribe('message:received', handler);

    bus.emit('message:received', {});
    expect(handler).toHaveBeenCalledTimes(1);

    unsub();
    bus.emit('message:received', {});
    expect(handler).toHaveBeenCalledTimes(1); // Still 1
  });

  it('assigns monotonic sequence numbers per session', async () => {
    const seqs: number[] = [];
    bus.subscribe('*', (e) => seqs.push(e.seq));

    await bus.emit('message:received', {}, { sessionKey: 'session-1' });
    await bus.emit('tool:before', {}, { sessionKey: 'session-1' });
    await bus.emit('tool:after', {}, { sessionKey: 'session-1' });

    expect(seqs).toEqual([1, 2, 3]);
  });
});
```

### B. Backend Contract Testing

Based on `src/agents/tools/memory-tool.citations.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import type { MemoriaBackend } from './types.js';

const createStubBackend = (overrides?: Partial<MemoriaBackend>): MemoriaBackend => ({
  id: 'test-backend',
  name: 'Test Backend',
  onRetain: vi.fn(async () => {}),
  onRecall: vi.fn(async () => []),
  onCompactionPre: vi.fn(async () => {}),
  onCompactionPost: vi.fn(async () => {}),
  ...overrides,
});

describe('MemoriaClient backend routing', () => {
  it('routes retain to active backends', async () => {
    const backend = createStubBackend();
    const client = createMemoriaClient({ backends: [backend] });

    await client.retain('user prefers TypeScript', { category: 'preference' });

    expect(backend.onRetain).toHaveBeenCalledWith(
      'user prefers TypeScript',
      expect.objectContaining({ category: 'preference' }),
    );
  });

  it('merges results from multiple backends', async () => {
    const backend1 = createStubBackend({
      id: 'backend-1',
      onRecall: vi.fn(async () => [{ content: 'result-1', score: 0.9, source: 'backend-1' }]),
    });
    const backend2 = createStubBackend({
      id: 'backend-2',
      onRecall: vi.fn(async () => [{ content: 'result-2', score: 0.8, source: 'backend-2' }]),
    });

    const client = createMemoriaClient({ backends: [backend1, backend2] });
    const results = await client.recall('preferences');

    expect(results).toHaveLength(2);
    expect(results[0].source).toBe('backend-1'); // Higher score first
  });

  it('falls back when primary backend fails', async () => {
    const primary = createStubBackend({
      id: 'primary',
      onRecall: vi.fn(async () => { throw new Error('connection failed'); }),
    });
    const fallback = createStubBackend({
      id: 'fallback',
      onRecall: vi.fn(async () => [{ content: 'fallback result', score: 0.7, source: 'fallback' }]),
    });

    const client = createMemoriaClient({
      backends: [primary, fallback],
      fallbackOnError: true,
    });

    const results = await client.recall('test');
    expect(results[0].source).toBe('fallback');
  });
});
```

### C. Compaction Timing Verification

Critical test for the core problem Memoria solves:

```typescript
describe('Compaction event timing', () => {
  it('fires compaction:pre BEFORE any data is pruned', async () => {
    const timeline: string[] = [];
    let capturedMessages: unknown[] = [];

    // Simulate session with messages
    const session = createTestSession({
      messages: [{ role: 'user', content: 'message 1' }, /* ... */],
    });

    memoria.on('compaction:pre', async (event) => {
      timeline.push('pre');
      capturedMessages = [...event.messages]; // Capture full state
    });

    session.onPrune(() => {
      timeline.push('prune');
    });

    memoria.on('compaction:post', () => {
      timeline.push('post');
    });

    await triggerCompaction(session);

    // CRITICAL: pre must fire before prune
    expect(timeline).toEqual(['pre', 'prune', 'post']);
    expect(capturedMessages.length).toBeGreaterThan(0);
  });

  it('waits for backend acks before proceeding', async () => {
    const backend = createStubBackend({
      onCompactionPre: vi.fn(async () => {
        await delay(100); // Simulate slow capture
      }),
    });

    const client = createMemoriaClient({
      backends: [backend],
      requireAck: true,
      ackTimeout: 5000,
    });

    const start = Date.now();
    await client.handleCompaction({ messages: [], tokenCount: 50000 });
    const elapsed = Date.now() - start;

    expect(elapsed).toBeGreaterThanOrEqual(100);
    expect(backend.onCompactionPre).toHaveBeenCalled();
  });
});
```

### D. Plugin Hook Integration

Based on existing hook patterns:

```typescript
describe('Memoria plugin hooks', () => {
  it('wires before_compaction to backend capture', async () => {
    const captured: unknown[] = [];
    const backend = createStubBackend({
      onCompactionPre: vi.fn(async (event) => {
        captured.push(event);
      }),
    });

    const plugin = createMemoriaPlugin({ backends: [backend] });
    const api = createTestPluginApi();

    plugin.register(api);

    // Simulate hook trigger
    await api.triggerHook('before_compaction', {
      messageCount: 100,
      tokenCount: 50000,
    });

    expect(captured).toHaveLength(1);
    expect(backend.onCompactionPre).toHaveBeenCalled();
  });

  it('injects recalled context on before_agent_start', async () => {
    const backend = createStubBackend({
      onRecall: vi.fn(async () => [
        { content: 'User prefers Python', score: 0.9, source: 'hindsight' },
      ]),
    });

    const plugin = createMemoriaPlugin({
      backends: [backend],
      autoRecall: true,
    });
    const api = createTestPluginApi();

    plugin.register(api);

    const result = await api.triggerHook('before_agent_start', {
      prompt: 'Help me write code',
    });

    expect(result.prependContext).toContain('User prefers Python');
  });
});
```

---

## 4. Test Utilities

Create `src/memoria/test-helpers.ts`:

```typescript
import { vi } from 'vitest';
import type { MemoriaBackend, MemoriaClient, MemoriaEvent } from './types.js';

/**
 * Create a stub backend with all methods mocked.
 */
export function createStubBackend(
  overrides?: Partial<MemoriaBackend>,
): MemoriaBackend {
  return {
    id: 'stub-backend',
    name: 'Stub Backend',
    onRetain: vi.fn(async () => {}),
    onRecall: vi.fn(async () => []),
    onReflect: vi.fn(async () => []),
    onCompactionPre: vi.fn(async () => {}),
    onCompactionPost: vi.fn(async () => {}),
    start: vi.fn(async () => {}),
    stop: vi.fn(async () => {}),
    healthCheck: vi.fn(async () => ({ ok: true })),
    ...overrides,
  };
}

/**
 * Create a test event bus that captures all emitted events.
 */
export function createTestEventBus(): {
  bus: MemoriaEventBus;
  events: MemoriaEvent[];
  getEventsByType: (type: string) => MemoriaEvent[];
} {
  const events: MemoriaEvent[] = [];
  const bus = createMemoriaEventBus({
    onEmit: (event) => events.push(event),
  });

  return {
    bus,
    events,
    getEventsByType: (type) => events.filter((e) => e.type === type),
  };
}

/**
 * Create a test plugin API for hook testing.
 */
export function createTestPluginApi(): TestPluginApi {
  const hooks = new Map<string, Array<(...args: unknown[]) => unknown>>();

  return {
    on: (name, handler) => {
      const list = hooks.get(name) ?? [];
      list.push(handler);
      hooks.set(name, list);
    },
    triggerHook: async (name, event, ctx = {}) => {
      const handlers = hooks.get(name) ?? [];
      let result = {};
      for (const handler of handlers) {
        const r = await handler(event, ctx);
        if (r) result = { ...result, ...r };
      }
      return result;
    },
    // ... other api methods
  };
}

/**
 * Simulate a compaction trigger for testing.
 */
export async function simulateCompaction(
  client: MemoriaClient,
  opts: {
    messages?: unknown[];
    tokenCount?: number;
    sessionKey?: string;
  },
): Promise<void> {
  await client.emit('compaction:pre', {
    messages: opts.messages ?? [],
    tokenCount: opts.tokenCount ?? 50000,
    sessionKey: opts.sessionKey ?? 'test-session',
  });
}

/**
 * Wait for async operations to settle.
 */
export function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

---

## 5. Development Phases with Test Gates

| Phase | Deliverable | Test Gate | Coverage Target |
|-------|-------------|-----------|-----------------|
| 1. Types | `src/memoria/types.ts` | TypeScript compiles | N/A |
| 2. Event Bus | `event-bus.ts` + `.test.ts` | Event ordering, sub/unsub, sequences | 90% |
| 3. Backend Interface | Contract tests with stubs | All methods callable | 90% |
| 4. Compaction Events | Hook wiring + timing tests | Pre fires BEFORE prune | 95% |
| 5. Local Backend | `backends/local.ts` | Fallback always works | 80% |
| 6. Hindsight Backend | `backends/hindsight.ts` + tests | Mocked + live (gated) | 80% |
| 7. Unified Client | `client.ts` + tests | Routing, fallback, fusion | 85% |
| 8. Plugin Integration | Extension structure | Hook registration works | 80% |
| 9. E2E Integration | `integration.e2e.test.ts` | Full flow with gateway | 70% |

### Phase Gate Commands

```bash
# Phase 1: Types compile
pnpm tsgo

# Phase 2-4: Unit tests pass
pnpm test src/memoria/

# Phase 5-7: Coverage threshold
pnpm test:coverage src/memoria/

# Phase 8: Extension tests
pnpm test extensions/memoria-*/

# Phase 9: E2E
pnpm test:e2e src/memoria/

# Live tests (optional, requires API keys)
MEMORIA_LIVE=1 pnpm test src/memoria/**/*.live.test.ts
```

---

## 6. Plugin Structure

Follow `extensions/memory-lancedb/` as the template:

### Extension Layout

```
extensions/memoria-hindsight/
├── package.json
├── openclaw.plugin.json
├── config.ts                   # Config schema + validation
├── index.ts                    # Plugin definition
├── index.test.ts               # Unit tests
└── index.live.test.ts          # Real API tests (gated)
```

### package.json

```json
{
  "name": "@openclaw/memoria-hindsight",
  "version": "0.0.1",
  "type": "module",
  "main": "index.ts",
  "dependencies": {
    "@anthropic-ai/tokenizer": "^0.0.3"
  },
  "devDependencies": {
    "openclaw": "workspace:*"
  },
  "peerDependencies": {
    "openclaw": "*"
  }
}
```

### openclaw.plugin.json

```json
{
  "id": "memoria-hindsight",
  "name": "Memoria Hindsight",
  "description": "Hindsight memory backend for Memoria",
  "kind": "memory",
  "version": "0.0.1"
}
```

### index.ts (Plugin Definition)

```typescript
import type { OpenClawPluginApi, OpenClawPluginDefinition } from 'openclaw/plugin-sdk';
import { configSchema } from './config.js';

const plugin: OpenClawPluginDefinition = {
  id: 'memoria-hindsight',
  name: 'Memoria Hindsight',
  description: 'Hindsight memory backend with world/experience/mental models',
  kind: 'memory',
  configSchema,

  async register(api: OpenClawPluginApi) {
    const config = api.pluginConfig as HindsightConfig;

    // Initialize Hindsight client
    const client = createHindsightClient(config);

    // Register compaction hooks
    api.on('before_compaction', async (event, ctx) => {
      await client.retain(serializeContext(event), {
        memoryBankId: ctx.agentId,
        metadata: { type: 'compaction_snapshot' },
      });
    });

    // Register auto-recall on agent start
    api.on('before_agent_start', async (event, ctx) => {
      if (!config.autoRecall) return;

      const memories = await client.search(event.prompt, { limit: 5 });
      if (memories.length === 0) return;

      return {
        prependContext: formatMemories(memories),
      };
    });

    // Register auto-capture on agent end
    api.on('agent_end', async (event, ctx) => {
      if (!config.autoCapture) return;

      const important = extractImportant(event.messages);
      if (important) {
        await client.retain(important, { memoryBankId: ctx.agentId });
      }
    });

    // Register tools
    api.registerTool((ctx) => ({
      name: 'memoria_recall',
      description: 'Search long-term memory',
      parameters: { query: { type: 'string' } },
      execute: async (id, { query }) => {
        const results = await client.search(query);
        return { results };
      },
    }));

    // Register CLI commands
    api.registerCli(({ program }) => {
      program
        .command('memoria:status')
        .description('Show Memoria backend status')
        .action(async () => {
          const health = await client.healthCheck();
          console.log(health.ok ? 'Healthy' : `Error: ${health.error}`);
        });
    });

    // Register service lifecycle
    api.registerService({
      id: 'memoria-hindsight',
      start: async () => {
        await client.connect();
        api.logger.info('Hindsight backend connected');
      },
      stop: async () => {
        await client.disconnect();
      },
    });
  },
};

export default plugin;
```

---

## 7. CI Integration

### Package.json Scripts

```json
{
  "scripts": {
    "test:memoria": "vitest run src/memoria/**/*.test.ts extensions/memoria-*/**/*.test.ts",
    "test:memoria:watch": "vitest watch src/memoria/",
    "test:memoria:coverage": "vitest run --coverage src/memoria/",
    "test:memoria:live": "MEMORIA_LIVE=1 vitest run src/memoria/**/*.live.test.ts extensions/memoria-*/**/*.live.test.ts"
  }
}
```

### Live Test Gating

```typescript
// In live test files
const describeLive = process.env.MEMORIA_LIVE === '1' &&
  process.env.HINDSIGHT_API_KEY
    ? describe
    : describe.skip;

describeLive('Hindsight live tests', () => {
  it('connects to real API', async () => {
    const client = createHindsightClient({
      apiKey: process.env.HINDSIGHT_API_KEY!,
    });

    const health = await client.healthCheck();
    expect(health.ok).toBe(true);
  });
});
```

---

## 8. Key Patterns from Codebase

### Dependency Injection

Follow `src/cli/deps.ts` pattern:

```typescript
export type MemoriaDeps = {
  createEventBus: typeof createMemoriaEventBus;
  createClient: typeof createMemoriaClient;
  backends: MemoriaBackend[];
};

export function createDefaultMemoriaDeps(): MemoriaDeps {
  return {
    createEventBus,
    createClient,
    backends: [createLocalBackend()],
  };
}
```

### Config Resolution

Follow `src/memory/backend-config.ts`:

```typescript
export function resolveMemoriaConfig(
  config: OpenClawConfig,
  agentId: string,
): ResolvedMemoriaConfig {
  const defaults = config.memoria?.defaults ?? {};
  const agentConfig = config.agents?.list?.find(a => a.id === agentId)?.memoria;

  return {
    ...defaults,
    ...agentConfig,
    backends: mergeBackends(defaults.backends, agentConfig?.backends),
  };
}
```

### Error Handling

Follow existing patterns - no over-engineering:

```typescript
// Good: Simple try/catch with fallback
try {
  return await primaryBackend.recall(query);
} catch (error) {
  logger.warn(`Primary backend failed: ${error.message}`);
  return fallbackBackend.recall(query);
}

// Avoid: Complex error hierarchies for simple cases
```

---

## 9. Documentation Checklist

When implementing, update:

- [ ] `docs/MEMORIA-FOUNDATION.md` - Architecture (already exists)
- [ ] `docs/memoria/events.md` - Event reference
- [ ] `docs/memoria/backends.md` - Backend configuration
- [ ] `docs/memoria/migration.md` - Migration from base OpenClaw
- [ ] `docs/testing.md` - Add Memoria test section

---

## Related Documents

- [OPENCLAW-MEMORIA-FOUNDATION.md](./OPENCLAW-MEMORIA-FOUNDATION.md) - Architecture vision
- [MEMORY-SYSTEM-ANALYSIS.md](./MEMORY-SYSTEM-ANALYSIS.md) - Current OpenClaw memory internals
- [MEMORY-COMMUNITY-RESEARCH.md](./MEMORY-COMMUNITY-RESEARCH.md) - Community solutions
- [MEMORY-EXTERNAL-SYSTEMS-COMPARISON.md](./MEMORY-EXTERNAL-SYSTEMS-COMPARISON.md) - External system comparison

---

*Document version: 1.0*
*Last updated: 2026-02-03*
