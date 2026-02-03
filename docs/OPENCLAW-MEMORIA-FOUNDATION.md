# OpenClaw Memoria: Foundation Document

> A fork of OpenClaw focused on robust, intelligent memory systems with proper event emission, external memory integration, and data loss prevention.

---

## Vision

**OpenClaw Memoria** transforms OpenClaw from a messaging gateway with basic memory into an **intelligent agent platform with true learning capabilities**.

Core principles:
1. **Never lose data** - Events emitted before any destructive operation
2. **Learn, don't just remember** - Integration with intelligent memory systems
3. **Multiple memory backends** - Hindsight, Graphiti, Mem0, NATS, and more
4. **Event-driven architecture** - Everything emits events for external systems

---

## Why This Fork?

### Current OpenClaw Memory Problems

Based on GitHub issues research, OpenClaw has critical memory-related problems:

#### 1. Silent Data Loss (#5429)

> "Lost ~45 hours of agent work/context due to compaction with no automatic save mechanism."

Users report losing days of context because:
- Compaction happens silently with no warning
- Nothing is saved to disk automatically
- No indication that compaction occurred or what was lost

#### 2. Hooks Exist But Never Fire (#6535)

Plugin hooks are defined in types but never called in execution flow:

| Hook | Status |
|------|--------|
| `before_compaction` | ❌ Not wired |
| `after_compaction` | ❌ Not wired |
| `before_tool_call` | ❌ Not wired |
| `after_tool_call` | ❌ Not wired |
| `message_sending` | ❌ Not wired |
| `message_sent` | ❌ Not wired |
| `session_start` | ❌ Not wired |
| `session_end` | ❌ Not wired |

Only 4/12 hooks actually work: `before_agent_start`, `agent_end`, `message_received`, `tool_result_persist`.

#### 3. Pre-Compaction Hook Fires Too Late (#6768)

Even the proposed fix has a critical flaw:

> "session:compact:pre currently fires AFTER history is already pruned/replaced, which undermines the primary use-case (capturing full pre-compaction state)"

The hook fires when data is already gone.

#### 4. Memory Flush Overwrites Files (#6877)

The default memory flush prompt causes agents to **overwrite** existing memory files instead of appending, losing earlier entries from the same day.

#### 5. Stale Token Counts (#5457)

Memory flush uses stale token counts, so it can be bypassed and compaction happens without the flush ever running.

### What Memoria Solves

| Problem | OpenClaw | Memoria |
|---------|----------|---------|
| Silent compaction | ❌ No warning | ✅ Events + external backup |
| Data loss | ❌ Common | ✅ NATS event log (replay) |
| Hooks not wired | ❌ 8/12 missing | ✅ All hooks fire |
| Pre-compact timing | ❌ Fires too late | ✅ Fires BEFORE prune |
| Memory overwrite | ❌ Overwrites files | ✅ Append-only + versioning |
| Learning | ❌ Compression only | ✅ Hindsight Reflect |
| Relationships | ❌ None | ✅ Graphiti knowledge graph |

---

## Architecture

### Event Flow

```
User message
     │
     ▼
┌────────────────────────────────────────────────────────────┐
│                     MEMORIA EVENT BUS                       │
│                   (NATS JetStream backbone)                 │
└────────────────────────────────────────────────────────────┘
     │
     ├──▶ message:received ──▶ Hindsight.retain()
     │
     ▼
Agent processing
     │
     ├──▶ tool:before ──▶ Log tool invocation
     ├──▶ tool:after ──▶ Log result
     │
     ▼
Context approaching limit
     │
     ├──▶ compaction:warning (80%) ──▶ Alert + memory systems prepare
     ├──▶ compaction:pre (BEFORE prune) ──▶ Full state capture
     │         │
     │         ├──▶ Hindsight.retain(full_context)
     │         ├──▶ Graphiti.snapshot()
     │         └──▶ NATS.publish(raw_events)
     │
     ▼
Compaction executes
     │
     ├──▶ compaction:post ──▶ Inject recovery context
     │         │
     │         ├──▶ Hindsight.recall(what_was_lost)
     │         └──▶ Graphiti.inject_relationships()
     │
     ▼
Agent continues with restored context
     │
     ├──▶ agent:end ──▶ Hindsight.reflect() + capture
     │
     ▼
Response delivered
```

### Memory Layer Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    MEMORIA UNIFIED API                       │
│              memoria.retain() / recall() / reflect()        │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   HINDSIGHT   │   │   GRAPHITI    │   │    MEM0       │
│               │   │               │   │               │
│ World/Exp/    │   │ Temporal      │   │ User/Session/ │
│ Mental Models │   │ Knowledge     │   │ Agent         │
│               │   │ Graph         │   │ Memories      │
│ + Reflect     │   │               │   │               │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   NATS JETSTREAM                             │
│              (Event backbone - raw event log)                │
│                                                              │
│   Enables: Time travel, Replay, Audit, Multi-agent          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 LOCAL MEMORY (Fallback)                      │
│          Markdown files + SQLite (existing OpenClaw)        │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Memoria Event Bus

All significant events published to NATS JetStream:

```typescript
type MemoriaEvent = {
  id: string;
  timestamp: number;
  agentId: string;
  sessionKey: string;
  type: MemoriaEventType;
  payload: unknown;
};

type MemoriaEventType =
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

  // Compaction lifecycle (CRITICAL)
  | 'compaction:warning'    // 80% context used
  | 'compaction:imminent'   // 95% context used
  | 'compaction:pre'        // BEFORE any pruning
  | 'compaction:executing'  // During compaction
  | 'compaction:post'       // After compaction
  | 'compaction:failed'     // Compaction error

  // Memory lifecycle
  | 'memory:write'
  | 'memory:search'
  | 'memory:flush';
```

### 2. Pre-Compaction Event System

The critical fix: emit events **BEFORE** any data loss.

```typescript
async function handleCompaction(session: Session) {
  const state = captureFullState(session);

  // FIRST: Emit warning events
  if (state.contextUsage >= 0.8) {
    await eventBus.emit('compaction:warning', {
      usage: state.contextUsage,
      messagesAtRisk: state.messageCount,
      tokensAtRisk: state.tokenCount,
    });
  }

  // SECOND: Emit pre-compaction with FULL context
  await eventBus.emit('compaction:pre', {
    sessionId: session.id,
    messages: session.getAllMessages(),  // Full, not pruned
    tools: session.getToolCalls(),
    context: session.getFullContext(),
    timestamp: Date.now(),
  });

  // THIRD: Wait for external systems to confirm capture
  await waitForAcks(['hindsight', 'graphiti', 'nats']);

  // FOURTH: Now safe to compact
  const result = await session.compact();

  // FIFTH: Emit post-compaction
  await eventBus.emit('compaction:post', {
    summary: result.summary,
    tokensBefore: state.tokenCount,
    tokensAfter: result.tokensAfter,
    messagesRemoved: result.removedCount,
  });

  // SIXTH: Inject recovery context from memory systems
  const recovery = await memoria.recall(state.keyTopics);
  await session.injectContext(recovery);
}
```

### 3. Unified Memory API

Single interface to all memory backends:

```typescript
interface MemoriaClient {
  // Core operations
  retain(content: string, metadata?: RetainOptions): Promise<void>;
  recall(query: string, options?: RecallOptions): Promise<MemoriaResult[]>;
  reflect(topic?: string): Promise<Insight[]>;  // Hindsight-specific

  // Relationship operations (Graphiti)
  addRelationship(entity1: string, relation: string, entity2: string): Promise<void>;
  queryRelationships(entity: string): Promise<Relationship[]>;
  temporalQuery(query: string, asOf?: Date): Promise<MemoriaResult[]>;

  // Event operations (NATS)
  subscribe(eventType: MemoriaEventType, handler: EventHandler): void;
  replay(since: Date, filter?: EventFilter): AsyncIterable<MemoriaEvent>;

  // Multi-backend query
  search(query: string, options?: SearchOptions): Promise<UnifiedResult[]>;
}
```

### 4. Memory Backend Plugins

#### Hindsight Plugin
```typescript
const hindsightPlugin: MemoriaPlugin = {
  id: 'memoria-hindsight',

  async onRetain(content, meta) {
    await hindsight.memories.add({
      content,
      memoryBankId: meta.agentId,
      metadata: { source: 'openclaw', ...meta }
    });
  },

  async onRecall(query, options) {
    return hindsight.memories.search({
      query,
      memoryBankId: options.agentId,
      topK: options.limit,
    });
  },

  async onReflect(topic) {
    // Unique to Hindsight: generate insights
    return hindsight.memories.reflect({
      topic,
      memoryBankId: options.agentId,
    });
  },

  async onCompactionPre(event) {
    // Capture full context before loss
    await hindsight.memories.add({
      content: serializeContext(event.messages),
      memoryBankId: event.agentId,
      metadata: {
        type: 'compaction_snapshot',
        messageCount: event.messages.length,
      }
    });
  },
};
```

#### Graphiti Plugin
```typescript
const graphitiPlugin: MemoriaPlugin = {
  id: 'memoria-graphiti',

  async onRetain(content, meta) {
    // Extract entities and relationships
    await graphiti.add_episode(
      content,
      new Date(),
      'message',
      { agentId: meta.agentId }
    );
  },

  async onRecall(query, options) {
    return graphiti.search(query, {
      num_results: options.limit,
    });
  },

  async onTemporalQuery(query, asOf) {
    // Graphiti's killer feature: point-in-time queries
    return graphiti.search(query, {
      as_of: asOf,
    });
  },

  async onCompactionPre(event) {
    // Snapshot relationship state
    const entities = await extractEntities(event.messages);
    for (const entity of entities) {
      await graphiti.add_episode(
        `Pre-compaction state: ${entity}`,
        new Date(),
        'compaction_snapshot'
      );
    }
  },
};
```

---

## Implementation Phases

### Phase 1: Event Foundation (Week 1-2)

**Goal:** Fix the broken hook system and add proper event emission.

Tasks:
1. Wire all 12 plugin hooks to actually fire
2. Add `compaction:warning` event at 80% context
3. Fix `compaction:pre` to fire BEFORE pruning (not after)
4. Add NATS JetStream as optional event backbone
5. Create `MemoriaEventBus` abstraction

Fixes issues: #6535, #6768, #5429

### Phase 2: NATS Integration (Week 2-3)

**Goal:** Port and validate PR #7358 (NATS JetStream).

Tasks:
1. Merge/adapt NATS event store PR
2. Add event replay capabilities
3. Add temporal queries via event log
4. Test multi-agent coordination
5. Document event schema

### Phase 3: Hindsight Integration (Week 3-4)

**Goal:** First external memory system integration.

Tasks:
1. Create `memoria-hindsight` plugin
2. Implement retain/recall/reflect
3. Wire to compaction events
4. Add auto-capture on `agent:end`
5. Add auto-recall on `agent:start`

### Phase 4: Graphiti Integration (Week 4-5)

**Goal:** Add relationship and temporal capabilities.

Tasks:
1. Create `memoria-graphiti` plugin
2. Implement entity/relationship extraction
3. Add temporal queries ("what changed since X")
4. Wire to compaction events
5. Test with Hindsight (complementary, not competing)

### Phase 5: Unified API & Polish (Week 5-6)

**Goal:** Single interface to all backends.

Tasks:
1. Create `MemoriaClient` unified API
2. Add intelligent routing (which backend for which query)
3. Add fallback chain (Hindsight → Graphiti → Local)
4. Performance optimization
5. Documentation and examples

---

## Configuration

```yaml
# memoria.yaml
memoria:
  enabled: true

  # Event backbone
  events:
    backend: nats  # or 'local' for in-process only
    nats:
      url: nats://localhost:4222
      stream: openclaw-memoria

  # Memory backends (can enable multiple)
  backends:
    hindsight:
      enabled: true
      apiUrl: https://api.vectorize.io/v1
      apiKey: ${HINDSIGHT_API_KEY}
      memoryBankId: ${AGENT_ID}
      autoCapture: true
      autoRecall: true
      reflect:
        enabled: true
        interval: 1h

    graphiti:
      enabled: true
      neo4jUrl: bolt://localhost:7687
      neo4jAuth: ${NEO4J_AUTH}
      extractRelationships: true
      temporalQueries: true

    mem0:
      enabled: false
      apiKey: ${MEM0_API_KEY}

    local:
      enabled: true  # Always available as fallback

  # Compaction protection
  compaction:
    warningThreshold: 0.8    # Emit warning at 80%
    imminentThreshold: 0.95  # Emit imminent at 95%
    requireAck: true         # Wait for backends before compacting
    ackTimeout: 5000         # ms to wait for acks

  # Unified search
  search:
    backends: [hindsight, graphiti, local]
    strategy: parallel  # or 'cascade'
    fusion: reciprocal_rank
```

---

## API Examples

### Basic Usage

```typescript
import { memoria } from 'openclaw-memoria';

// Store a memory
await memoria.retain('User prefers Python with type hints', {
  category: 'preference',
  confidence: 0.9,
});

// Recall memories
const results = await memoria.recall('What programming languages does user like?');

// Generate insights (Hindsight)
const insights = await memoria.reflect('user preferences');

// Temporal query (Graphiti)
const pastState = await memoria.temporalQuery(
  'What was the project status?',
  new Date('2025-01-15')
);

// Replay events (NATS)
for await (const event of memoria.replay(new Date('2025-01-01'))) {
  console.log(event.type, event.payload);
}
```

### Compaction Hook

```typescript
// Register for compaction events
memoria.on('compaction:pre', async (event) => {
  console.log(`Compaction imminent! ${event.messagesAtRisk} messages at risk`);

  // Custom backup logic
  await customBackup(event.messages);
});

memoria.on('compaction:post', async (event) => {
  console.log(`Compaction complete. Tokens: ${event.tokensBefore} → ${event.tokensAfter}`);

  // Inject recovery context
  const recovery = await memoria.recall(event.summary);
  return { injectContext: formatRecovery(recovery) };
});
```

---

## Migration from OpenClaw

### For Users

```bash
# Install memoria fork
npm install -g openclaw-memoria

# Migrate config
openclaw-memoria migrate --from ~/.openclaw

# Enable memory backends
openclaw-memoria config set memoria.backends.hindsight.enabled true
openclaw-memoria config set memoria.backends.hindsight.apiKey $HINDSIGHT_KEY

# Start gateway with memoria
openclaw-memoria gateway run
```

### For Plugin Developers

```typescript
// Before (OpenClaw)
api.on('before_compaction', handler);  // Never fires!

// After (Memoria)
api.on('compaction:pre', handler);  // Always fires, BEFORE pruning
```

---

## Success Metrics

| Metric | OpenClaw | Memoria Target |
|--------|----------|----------------|
| Data loss incidents | Common | Zero |
| Hooks firing | 4/12 | 12/12 |
| Pre-compact capture | After prune | Before prune |
| Memory backends | 2 (SQLite, LanceDB) | 5+ (+ Hindsight, Graphiti, Mem0, NATS) |
| Temporal queries | No | Yes |
| Relationship understanding | No | Yes |
| Learning (Reflect) | No | Yes |
| Event replay | No | Yes |

---

## Repository Structure

```
openclaw-memoria/
├── src/
│   ├── memoria/
│   │   ├── event-bus.ts          # Event emission layer
│   │   ├── client.ts             # Unified MemoriaClient
│   │   ├── backends/
│   │   │   ├── hindsight.ts
│   │   │   ├── graphiti.ts
│   │   │   ├── mem0.ts
│   │   │   ├── nats.ts
│   │   │   └── local.ts
│   │   ├── compaction/
│   │   │   ├── events.ts         # Pre/post compaction events
│   │   │   ├── protection.ts     # Data loss prevention
│   │   │   └── recovery.ts       # Context recovery
│   │   └── types.ts
│   └── ... (rest of OpenClaw)
├── extensions/
│   ├── memoria-hindsight/
│   ├── memoria-graphiti/
│   ├── memoria-mem0/
│   └── memoria-nats/
├── docs/
│   ├── MEMORIA-FOUNDATION.md     # This document
│   ├── MEMORIA-EVENTS.md         # Event reference
│   ├── MEMORIA-BACKENDS.md       # Backend configuration
│   └── MEMORIA-MIGRATION.md      # Migration guide
└── examples/
    ├── basic-usage/
    ├── multi-backend/
    └── compaction-protection/
```

---

## Next Steps

1. **Create fork repository** - `openclaw-memoria`
2. **Fix hook system** - Wire all 12 hooks
3. **Fix compaction timing** - Pre-event before prune
4. **Port NATS PR** - Event backbone
5. **Build Hindsight plugin** - First external backend
6. **Build Graphiti plugin** - Relationships + temporal
7. **Create unified API** - Single interface
8. **Documentation** - Migration guide + examples

---

## Related Documents

- [MEMORY-SYSTEM-ANALYSIS.md](./MEMORY-SYSTEM-ANALYSIS.md) - OpenClaw memory internals
- [MEMORY-COMMUNITY-RESEARCH.md](./MEMORY-COMMUNITY-RESEARCH.md) - Community solutions
- [MEMORY-EXTERNAL-SYSTEMS-COMPARISON.md](./MEMORY-EXTERNAL-SYSTEMS-COMPARISON.md) - External system comparison

---

*Document version: 1.0*
*Last updated: 2026-02-03*
