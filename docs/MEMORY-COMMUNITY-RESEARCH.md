# OpenClaw Memory: Community Solutions Research

> Research on existing memory plugins, PRs, and community requests in the OpenClaw ecosystem.
> Last updated: 2026-02-03

---

## Summary

The OpenClaw community has been actively working on memory improvements. Key findings:

1. **5+ memory plugins/PRs** in various stages of development
2. **Multiple architectural approaches**: hierarchical memory, event sourcing, external services
3. **Common pain points**: memory persistence, compaction data loss, local-first setup
4. **External services explored**: PowerMem, Redis/agent-memory-server, Supermemory, NATS JetStream
5. **No Hindsight/Graffiti integrations found** - opportunity for novel contribution

---

## Active PRs & Proposed Plugins

### 1. memory-redis (PR #8272) - Redis + agent-memory-server

**Author**: @abrookins (Andrew Brookins)
**Status**: OPEN
**Maturity**: Ready for review (772 LOC main file)

**Integration**: [agent-memory-server](https://github.com/redis-developer/agent-memory-server)

**Features**:
- Auto-recall: Continually refreshed user summary from long-term memory
- Auto-capture: Saves conversations to working memory for background extraction
- Manual tools: `memory_recall`, `memory_store`, `memory_forget`
- Configurable extraction strategies: discrete, summary, preferences, custom
- Multi-tenancy: namespace + userId isolation

**Files**:
```
extensions/memory-redis/
├── config.ts          (343 lines)
├── index.ts           (772 lines)
├── index.test.ts      (522 lines)
├── openclaw.plugin.json
└── package.json
```

**Dependencies**: `agent-memory-client ^0.3.1`

**Review Notes** (Greptile):
- Multi-tenancy safety concern: delete path bypasses namespace/user scoping
- Confidence: 3/5

---

### 2. CoreMemories (PR #7480) - Hierarchical Memory System

**Author**: @Itslouisbaby (Louis Angeli)
**Status**: OPEN
**Maturity**: Significant implementation (1050+ LOC)

**Architecture**: 3-layer hierarchical system inspired by human memory

```
HOT LAYER (Always loaded, ~1400 tokens)
├── Flash (0-48h): Full conversation detail
└── Warm (2-7d): Summaries + key quotes

RECENT LAYER (Triggered load)
└── Week 1-4: Progressive compression (50% → 20%)

ARCHIVE LAYER (Deep retrieval)
└── Fresh → Core (1 month → 1+ years): Essence extraction
```

**Key Features**:
- Local-first: Hot/Warm processing uses local Ollama (free, private, fast)
- Token efficient: ~30-40% reduction in memory tokens
- Event-driven: Load memories only when keywords match
- Progressive compression: Detail fades with age (human-like)
- HEARTBEAT integration: Compress Flash → Warm every 6 hours
- CRON integration: Smart reminders with context

**Configuration**:
```json
{
  "coreMemories": {
    "enabled": true,
    "compression": "auto",
    "engines": {
      "local": {
        "provider": "ollama",
        "model": "phi3:mini"
      }
    }
  }
}
```

**Files**:
```
packages/core-memories/
├── src/
│   ├── index.ts           (1050 lines)
│   └── integration.ts     (229 lines)
├── docs/
│   ├── spec.md            (457 lines)
│   └── architecture.md    (371 lines)
└── __tests__/
```

**Review Notes** (Greptile):
- Retrieval correctness bug: Warm entries in older week files not discoverable via keyword
- Config deep merge is shape-unsafe
- Confidence: 2/5 (needs fixes)

---

### 3. Event-Sourced Memory with NATS JetStream (PR #7358)

**Author**: @alberthild
**Status**: OPEN
**Maturity**: Production-validated (157K+ events, 30 days uptime)

**Architecture**: True event sourcing for AI agents

**Killer Features**:
1. **Temporal Queries** - Query any point in time
   ```
   "What did we decide about the API last Tuesday?"
   "Show me the context from when we started this project"
   ```

2. **Event Replay** - Full state reconstruction
   ```typescript
   await replayEvents({ since: "2024-01-01", agent: "main" });
   ```

3. **Projections** - Build any view from events
   - Learning system (extract preferences)
   - Analytics dashboard
   - Fact extractor (knowledge graph)
   - Compliance reports

4. **Multi-Agent Streams** - Isolated yet connected
5. **Real-time Subscriptions** - External tools can subscribe
6. **Self-hosted** - No cloud dependencies

**Production Stats**:
| Metric | Value |
|--------|-------|
| Events Processed | 157,000+ |
| Agents Coordinated | 5 |
| Uptime | 30 days |
| Storage | 165 MB |
| Latency | <5ms |

**Quick Start**:
```yaml
# gateway.yaml
eventStore:
  enabled: true
  natsUrl: nats://localhost:4222
```

**Files**:
```
src/infra/event-store.ts       (279 lines)
src/infra/event-context.ts     (330 lines)
src/infra/event-store.test.ts  (204 lines)
docs/features/event-store.md   (253 lines)
```

---

### 4. PowerMem Integration (Issue #7021 + PR #3842)

**Author**: @Teingi (jingshun.tq)
**Status**: OPEN (Issue) + PR exists
**Integration**: [PowerMem](https://github.com/oceanbase/powermem)

**Motivation**: Replace local SQLite with PowerMem's hybrid retrieval

**PowerMem Features**:
- Hybrid retrieval (dense + full-text + graph)
- Intelligent extraction/dedup/conflict resolution
- Multi-agent isolation and sharing primitives

**Proposed Approach**: "File-backed, PowerMem-powered search"
- Keep Markdown as canonical surface
- Use PowerMem for retrieval only
- Map results back to workspace paths + line numbers

**Plugin Structure**:
```
memory-powermem/
├── kind: "memory"
├── provides:
│   ├── memory_search (required)
│   ├── memory_get (required)
│   └── CLI parity (recommended)
```

---

### 5. WM/STM/LTM Memory Improvements (PR #7894)

**Author**: @bornio (Alex Vaystikh)
**Status**: OPEN
**Focus**: "Give OpenClaw better memory + REM sleep"

**Changes**:
- Auto-creates `STM.md` (Short-Term Memory) and `WORKING.md`
- Keeps LTM opt-in via `ltm/index.md` or `ltm/nodes/`
- Pre-compaction memory flush maintains these files

**Memory Layout**:
```
workspace/
├── WORKING.md      (readable, NOT indexed - keeps search high-signal)
├── STM.md          (indexed, auto-maintained)
├── MEMORY.md       (legacy, still supported)
├── memory/         (daily logs, still supported)
└── ltm/            (opt-in long-term memory)
    ├── index.md
    └── nodes/
```

**User Impact**:
- Preferences and decisions reliably land in durable memory
- `memory_search` retrieves personalization signals without extra config
- "It remembers how I like things" behavior improves over time

---

### 6. Voyage AI Embedding Support (PR #7078)

**Author**: @mcinteerj (Jake)
**Status**: OPEN
**Focus**: Native Voyage AI embeddings for memory search

**Features**:
- New `voyage` provider option
- Batch processing via Voyage Batch API
- File-upload + batch creation + polling + NDJSON streaming

**Files**:
```
src/memory/embeddings-voyage.ts     (92 lines)
src/memory/batch-voyage.ts          (374 lines)
src/memory/embeddings-voyage.test.ts
src/memory/batch-voyage.test.ts
```

**Why Voyage**:
- Better retrieval quality for some use cases
- Groundwork for future local execution

---

## Feature Requests & Issues

### Memory Architecture

| Issue | Title | Status |
|-------|-------|--------|
| #8185 | Memory flush on /new and /reset (pre-reset save) | OPEN |
| #7776 | Channel-aware memory context | OPEN |
| #7707 | Memory Trust Tagging by Source | OPEN |
| #7629 | memory-lancedb: Add hybrid search (BM25 + vector) | OPEN |
| #7618 | Post-compaction prompt injection | OPEN |
| #7175 | Pre-compaction hook for memory preservation | OPEN |
| #6622 | Pre-compaction memory flush with structured context | OPEN |

### External Services

| Issue | Title | Status |
|-------|-------|--------|
| #7021 | PowerMem integration | OPEN (has PR) |
| #6774 | Supermemory forget tool broken | OPEN |
| #8118 | memory-lancedb custom embedding providers | OPEN |
| #6668 | Gemini embedding support for LanceDB | OPEN |

### Bugs & Pain Points

| Issue | Title | Impact |
|-------|-------|--------|
| #6877 | Pre-compaction flush overwrites memory files | Data loss |
| #5429 | Lost 2 days of context to silent compaction | Data loss |
| #4868 | Memory index dirty, search returns empty | Retrieval broken |
| #7273 | status reports memory unavailable with memory-lancedb | UX |
| #8131 | memory tools require API key even with local provider | UX |
| #8217 | Memory indexing not populating chunks table | Setup |

---

## Existing Bundled Plugins

### memory-core (Default)

**Location**: `extensions/memory-core/`

**Provides**:
- `memory_search` tool
- `memory_get` tool
- CLI commands

**Backend**: Builtin SQLite with hybrid search (BM25 + Vector)

### memory-lancedb

**Location**: `extensions/memory-lancedb/`

**Provides**:
- `memory_recall` tool
- `memory_store` tool
- `memory_forget` tool
- Auto-recall via `before_agent_start` hook
- Auto-capture via `agent_end` hook

**Backend**: LanceDB with OpenAI embeddings

**Community Requests**:
- Hybrid search (#7629, PR #7636)
- Custom embedding providers (#8118)
- Gemini embedding support (#6668)

---

## Gaps & Opportunities

### Not Yet Addressed

1. **Hindsight** - No integration found
2. **Graffiti** - No integration found
3. **Mem0** - Only mentioned, no implementation
4. **Zep** - Only mentioned in comparison tables
5. **LangMem** - No integration found
6. **Letta (MemGPT)** - No integration found

### Common Community Pain Points

1. **Memory persistence** - Data loss during compaction
2. **Local-first setup** - API keys required even for local embeddings
3. **Memory organization** - No clear hierarchy or aging
4. **Cross-session continuity** - "Bot forgets things"
5. **Multi-agent memory** - Isolation and sharing

### Architectural Patterns Emerging

1. **Hierarchical aging** (CoreMemories) - Flash → Warm → Archive
2. **Event sourcing** (NATS) - Time travel + replay + projections
3. **External service** (Redis, PowerMem) - Offload to specialized systems
4. **File-backed + powered search** - Markdown source of truth + external retrieval

---

## Integration Strategy Recommendations

### For Hindsight/Graffiti

Based on community patterns, recommended approach:

**Option 1: Plugin Extension** (like memory-lancedb, memory-redis)
```
extensions/memory-hindsight/
├── index.ts
│   ├── registerTool("hindsight_search", ...)
│   ├── registerTool("hindsight_store", ...)
│   ├── on("before_agent_start", ...) // auto-recall
│   └── on("agent_end", ...) // auto-capture
├── config.ts
├── openclaw.plugin.json
└── package.json
```

**Option 2: Parallel Memory Layer** (like PowerMem proposal)
- Keep Markdown as source of truth
- Use external service for enhanced retrieval
- Map results back to workspace paths

**Option 3: Event-Sourced Bridge** (inspired by NATS PR)
- Stream events to Hindsight/Graffiti
- Build projections for different views
- Enable temporal queries

### Key Considerations

1. **Follow memory-lancedb patterns** - Well-established plugin structure
2. **Use lifecycle hooks** - `before_agent_start`, `agent_end`
3. **Implement fallback** - Graceful degradation if service unavailable
4. **Support multi-tenancy** - agentId/sessionKey scoping
5. **Include CLI commands** - `openclaw hindsight search/store/forget`

---

## Related Links

- PowerMem: https://github.com/oceanbase/powermem
- agent-memory-server: https://github.com/redis-developer/agent-memory-server
- QMD: https://github.com/tobi/qmd
- LanceDB: https://lancedb.github.io/lancedb/
- Voyage AI: https://www.voyageai.com/

---

*Document version: 1.0*
*Based on research from openclaw/openclaw GitHub repository*
