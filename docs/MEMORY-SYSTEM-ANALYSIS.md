# OpenClaw Memory System Analysis

> Research document for developing custom memory integration solutions (Hindsight, Graffiti, etc.)

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Memory Storage Model](#memory-storage-model)
3. [Memory Backends](#memory-backends)
4. [Embedding Providers](#embedding-providers)
5. [Search Mechanics](#search-mechanics)
6. [Agent Tools](#agent-tools)
7. [Plugin System](#plugin-system)
8. [Configuration Reference](#configuration-reference)
9. [Integration Strategies](#integration-strategies)
10. [File Locations](#file-locations)
11. [CLI Commands](#cli-commands)
12. [Development Notes](#development-notes)

---

## Architecture Overview

OpenClaw implements a layered memory architecture where **Markdown files are the source of truth** and various backends provide indexing and retrieval capabilities.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Agent Runtime                                │
│                    (Pi Agent via RPC mode)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Tools: memory_search, memory_get                                  │
│   Location: src/agents/tools/memory-tool.ts                         │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Memory Search Manager (src/memory/search-manager.ts)              │
│   - Factory for backend selection                                   │
│   - Automatic fallback (QMD → Builtin)                             │
│   - Caching layer                                                   │
│                                                                      │
├───────────────────────────┬─────────────────────────────────────────┤
│                           │                                          │
│   Builtin Backend         │   QMD Backend (experimental)            │
│   (src/memory/manager.ts) │   (src/memory/qmd-manager.ts)           │
│                           │                                          │
│   - SQLite + sqlite-vec   │   - External sidecar process            │
│   - FTS5 for BM25         │   - BM25 + Vector + Reranking           │
│   - Hybrid search         │   - Local GGUF models                   │
│   - 2355 LOC              │   - 810 LOC                             │
│                           │                                          │
├───────────────────────────┴─────────────────────────────────────────┤
│                                                                      │
│   Embedding Providers (src/memory/embeddings*.ts)                   │
│   - OpenAI (text-embedding-3-small, batch API)                      │
│   - Gemini (gemini-embedding-001, batch API)                        │
│   - Local (node-llama-cpp, embeddinggemma-300M)                     │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Source of Truth: Markdown Files                                   │
│   - MEMORY.md (long-term, curated, private sessions only)           │
│   - memory/YYYY-MM-DD.md (daily append-only logs)                   │
│   - Extra paths (configurable)                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Write Path**: Agent writes to Markdown files → File watcher detects change → Index updated
2. **Read Path**: Agent calls `memory_search` → Backend performs hybrid search → Results returned with snippets
3. **Compaction**: Before context compaction, `memoryFlush` triggers agent to persist important memories

---

## Memory Storage Model

### File-Based Source of Truth

OpenClaw treats **Markdown files as the canonical memory store**. The index is derived, not primary.

| File | Purpose | Loaded When |
|------|---------|-------------|
| `MEMORY.md` | Long-term curated memory | Main private session only |
| `memory/YYYY-MM-DD.md` | Daily logs (append-only) | Today + yesterday at session start |
| Custom paths | Additional indexed content | Via `memorySearch.extraPaths` |

**Default workspace**: `~/clawd/` (configurable via `agents.defaults.workspace`)

### Why Markdown?

- Human-readable and editable
- Version controllable (git)
- Portable across systems
- Agent can read/write without special tooling
- No vendor lock-in

---

## Memory Backends

### Builtin Backend (Default)

**File**: `src/memory/manager.ts` (2355 LOC)

**Storage**: SQLite database per agent at `~/.openclaw/memory/<agentId>.sqlite`

**Schema Tables**:
```sql
-- Metadata about the index
meta (key TEXT PRIMARY KEY, value TEXT)

-- Tracked source files
files (path TEXT PRIMARY KEY, hash TEXT, mtime INTEGER, size INTEGER)

-- Text chunks with embeddings
chunks (
  id TEXT PRIMARY KEY,
  path TEXT,
  startLine INTEGER,
  endLine INTEGER,
  text TEXT,
  embedding BLOB,  -- Float32Array serialized
  source TEXT      -- "memory" | "sessions"
)

-- Full-text search (FTS5)
chunks_fts (text)

-- Vector search (sqlite-vec extension)
chunks_vec (embedding FLOAT[dims])

-- Embedding cache
embedding_cache_v1 (provider TEXT, model TEXT, hash TEXT, embedding BLOB)
```

**Features**:
- Hybrid search (BM25 + Vector)
- File watching with 1.5s debounce
- Session transcript indexing (optional)
- Embedding cache to avoid re-computation
- Batch embedding support (OpenAI/Gemini)

### QMD Backend (Experimental)

**File**: `src/memory/qmd-manager.ts` (810 LOC)

**External dependency**: [QMD](https://github.com/tobi/qmd) CLI binary

**State directory**: `~/.openclaw/agents/<agentId>/qmd/`

**Enable**: Set `memory.backend = "qmd"` in config

**Advantages**:
- More sophisticated ranking (BM25 + Vector + Reranking)
- Local-only operation via Bun + node-llama-cpp
- Auto-downloads GGUF models from HuggingFace

**Configuration** (`memory.qmd.*`):
- `command` - QMD executable path (default: `qmd`)
- `includeDefaultMemory` - Auto-index workspace files (default: true)
- `paths[]` - Additional directories with glob patterns
- `sessions` - Transcript indexing options
- `update` - Refresh cadence (interval, debounce, onBoot)
- `limits` - Result caps (maxResults, maxSnippetChars, maxInjectedChars)

---

## Embedding Providers

### Provider Selection

**Auto-selection order** (when `provider: "auto"`):
1. Local (if `modelPath` configured and file exists)
2. OpenAI (if API key available)
3. Gemini (if API key available)
4. Disabled (until configured)

### OpenAI Embeddings

**File**: `src/memory/embeddings-openai.ts`

**Default model**: `text-embedding-3-small`

**Batch API**:
- Enabled by default
- Up to 50K requests per batch
- 24-hour processing window
- Discounted pricing

**Configuration**:
```json5
{
  memorySearch: {
    provider: "openai",
    model: "text-embedding-3-small",
    remote: {
      baseUrl: "https://api.openai.com/v1/",  // or custom endpoint
      apiKey: "sk-...",
      batch: { enabled: true, concurrency: 2 }
    }
  }
}
```

### Gemini Embeddings

**File**: `src/memory/embeddings-gemini.ts`

**Default model**: `gemini-embedding-001`

**Configuration**:
```json5
{
  memorySearch: {
    provider: "gemini",
    model: "gemini-embedding-001",
    remote: {
      apiKey: "..."  // or use GEMINI_API_KEY env var
    }
  }
}
```

### Local Embeddings

**File**: `src/memory/embeddings-local.ts`

**Default model**: `hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf` (~0.6 GB)

**Requirements**:
- `node-llama-cpp` native build
- Run `pnpm approve-builds` then `pnpm rebuild node-llama-cpp`

**Configuration**:
```json5
{
  memorySearch: {
    provider: "local",
    local: {
      modelPath: "hf:ggml-org/...",  // or local file path
      modelCacheDir: "~/.cache/openclaw/models/"
    },
    fallback: "none"  // don't fall back to remote
  }
}
```

---

## Search Mechanics

### Hybrid Search Algorithm

**File**: `src/memory/hybrid.ts`

The default search combines **BM25 (keyword)** and **Vector (semantic)** results:

```
1. Retrieve candidates from both:
   - Vector: top (maxResults × candidateMultiplier) by cosine similarity
   - BM25: top (maxResults × candidateMultiplier) by FTS5 rank

2. Convert BM25 rank to score:
   textScore = 1 / (1 + max(0, bm25Rank))

3. Merge by chunk ID with weighted score:
   finalScore = vectorWeight × vectorScore + textWeight × textScore

4. Return top maxResults by finalScore
```

**Default weights**: 70% vector, 30% text

**Why hybrid?**

| Query Type | Best Retriever |
|------------|----------------|
| "Mac Studio gateway host" | Vector (semantic match) |
| "a828e60" (commit hash) | BM25 (exact match) |
| "memorySearch.query.hybrid" | BM25 (config path) |
| "where do errors come from" | Vector (paraphrase) |

### Vector Search Implementation

**With sqlite-vec** (preferred):
```sql
SELECT id, distance FROM chunks_vec
WHERE embedding MATCH ?
ORDER BY distance
LIMIT ?
```

**Fallback** (no extension):
- Load all embeddings into memory
- Compute cosine similarity in JavaScript
- Sort and return top-k

### Chunking Strategy

**Default parameters**:
- Target: 400 tokens per chunk
- Overlap: 80 tokens

**Process**:
1. Parse Markdown with line tracking
2. Split into token-bounded chunks
3. Store with startLine/endLine for citations

---

## Agent Tools

### memory_search

**File**: `src/agents/tools/memory-tool.ts:25-88`

**Purpose**: Semantic search over indexed memory files

**Schema**:
```typescript
{
  query: string,       // Required: search query
  maxResults?: number, // Optional: max results (default from config)
  minScore?: number    // Optional: minimum relevance score
}
```

**Returns**:
```typescript
{
  results: [{
    path: string,      // File path (workspace-relative)
    startLine: number,
    endLine: number,
    score: number,     // 0-1 relevance
    snippet: string,   // ~700 chars of context
    source: "memory" | "sessions",
    citation?: string  // "path#L10-L15" format
  }],
  provider: string,
  model?: string,
  fallback?: { from: string, reason: string },
  citations: "auto" | "on" | "off"
}
```

### memory_get

**File**: `src/agents/tools/memory-tool.ts:90-135`

**Purpose**: Read specific lines from a memory file (after search identifies relevant sections)

**Schema**:
```typescript
{
  path: string,    // Required: workspace-relative path
  from?: number,   // Optional: starting line
  lines?: number   // Optional: number of lines
}
```

**Security**: Only allows paths matching `MEMORY.md` or `memory/**/*.md`

---

## Plugin System

### Plugin API

**File**: `src/plugins/types.ts`

**Core interface**: `OpenClawPluginApi`

```typescript
interface OpenClawPluginApi {
  // Identity
  id: string;
  name: string;
  config: OpenClawConfig;
  pluginConfig?: Record<string, unknown>;
  logger: PluginLogger;
  runtime: PluginRuntime;

  // Registration methods
  registerTool(tool: AnyAgentTool | ToolFactory, opts?): void;
  registerHook(events: string[], handler, opts?): void;
  registerCli(registrar, opts?): void;
  registerService(service): void;
  registerCommand(command): void;
  registerHttpRoute(params): void;
  registerChannel(registration): void;
  registerProvider(provider): void;
  registerGatewayMethod(method, handler): void;

  // Lifecycle hooks
  on<K extends PluginHookName>(hookName: K, handler, opts?): void;

  // Utilities
  resolvePath(input: string): string;
}
```

### Lifecycle Hooks

| Hook | When | Use Case |
|------|------|----------|
| `before_agent_start` | Before agent processes prompt | Inject context, modify system prompt |
| `agent_end` | After agent completes | Capture memories, log analytics |
| `before_compaction` | Before context compaction | Preserve important data |
| `after_compaction` | After context compaction | Update indices |
| `before_tool_call` | Before tool executes | Modify params, block calls |
| `after_tool_call` | After tool completes | Log results, trigger actions |
| `session_start` | New session begins | Initialize state |
| `session_end` | Session ends | Cleanup, persist data |

### Memory Plugin Example

**File**: `extensions/memory-lancedb/index.ts`

Key patterns from the LanceDB plugin:

```typescript
const memoryPlugin = {
  id: "memory-lancedb",
  kind: "memory",
  configSchema: memoryConfigSchema,

  register(api: OpenClawPluginApi) {
    const cfg = memoryConfigSchema.parse(api.pluginConfig);
    const db = new MemoryDB(cfg.dbPath);
    const embeddings = new Embeddings(cfg.embedding.apiKey);

    // 1. Register tools
    api.registerTool({
      name: "memory_recall",
      description: "Search long-term memories",
      parameters: { query: Type.String() },
      async execute(_id, params) {
        const vector = await embeddings.embed(params.query);
        const results = await db.search(vector);
        return { content: [{ type: "text", text: formatResults(results) }] };
      }
    });

    // 2. Auto-recall before agent starts
    api.on("before_agent_start", async (event) => {
      const vector = await embeddings.embed(event.prompt);
      const memories = await db.search(vector, 3, 0.3);
      if (memories.length > 0) {
        return {
          prependContext: `<relevant-memories>\n${formatMemories(memories)}\n</relevant-memories>`
        };
      }
    });

    // 3. Auto-capture after agent ends
    api.on("agent_end", async (event) => {
      if (!event.success) return;
      const capturable = extractCapturable(event.messages);
      for (const text of capturable) {
        const vector = await embeddings.embed(text);
        await db.store({ text, vector, category: detectCategory(text) });
      }
    });

    // 4. CLI commands
    api.registerCli(({ program }) => {
      program.command("ltm search <query>").action(async (query) => {
        const results = await db.search(await embeddings.embed(query));
        console.log(JSON.stringify(results, null, 2));
      });
    });
  }
};
```

### MemorySearchManager Interface

**File**: `src/memory/types.ts:61-80`

To create a custom backend, implement this interface:

```typescript
interface MemorySearchManager {
  // Core search functionality
  search(
    query: string,
    opts?: { maxResults?: number; minScore?: number; sessionKey?: string }
  ): Promise<MemorySearchResult[]>;

  // Read specific file content
  readFile(params: {
    relPath: string;
    from?: number;
    lines?: number;
  }): Promise<{ text: string; path: string }>;

  // Status reporting
  status(): MemoryProviderStatus;

  // Index synchronization
  sync?(params?: {
    reason?: string;
    force?: boolean;
    progress?: (update: MemorySyncProgressUpdate) => void;
  }): Promise<void>;

  // Health checks
  probeEmbeddingAvailability(): Promise<MemoryEmbeddingProbeResult>;
  probeVectorAvailability(): Promise<boolean>;

  // Cleanup
  close?(): Promise<void>;
}
```

---

## Configuration Reference

### Full Memory Configuration

```json5
{
  // Top-level memory settings
  memory: {
    backend: "builtin" | "qmd",  // Default: "builtin"
    citations: "auto" | "on" | "off",  // Default: "auto"

    // QMD backend settings (when backend: "qmd")
    qmd: {
      command: "qmd",  // Path to QMD binary
      includeDefaultMemory: true,  // Index MEMORY.md + memory/*.md
      paths: [
        { name: "docs", path: "~/notes", pattern: "**/*.md" }
      ],
      sessions: {
        enabled: false,
        retentionDays: 30,
        exportDir: "~/.openclaw/agents/{agentId}/qmd/sessions/"
      },
      update: {
        interval: "5m",
        debounceMs: 15000,
        onBoot: true,
        embedInterval: "5m"
      },
      limits: {
        maxResults: 6,
        maxSnippetChars: 700,
        maxInjectedChars: 4000,
        timeoutMs: 4000
      },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }]
      }
    }
  },

  agents: {
    defaults: {
      // Memory search settings
      memorySearch: {
        enabled: true,
        provider: "auto" | "openai" | "gemini" | "local",
        model: "text-embedding-3-small",  // Provider-specific
        fallback: "openai" | "gemini" | "local" | "none",
        sources: ["memory", "sessions"],
        extraPaths: [],  // Additional directories to index

        // Experimental features
        experimental: {
          sessionMemory: false  // Index session transcripts
        },

        // Remote embedding provider
        remote: {
          baseUrl: "https://api.openai.com/v1/",
          apiKey: "...",
          headers: {},
          batch: {
            enabled: true,
            wait: true,
            concurrency: 2,
            pollIntervalMs: 5000,
            timeoutMinutes: 30
          }
        },

        // Local embedding provider
        local: {
          modelPath: "hf:ggml-org/embeddinggemma-300M-GGUF/...",
          modelCacheDir: "~/.cache/openclaw/models/"
        },

        // SQLite store settings
        store: {
          driver: "sqlite",
          path: "~/.openclaw/memory/{agentId}.sqlite",
          vector: {
            enabled: true,
            extensionPath: null  // Auto-detect sqlite-vec
          }
        },

        // Chunking parameters
        chunking: {
          tokens: 400,
          overlap: 80
        },

        // Sync triggers
        sync: {
          onSessionStart: true,
          onSearch: true,
          watch: true,
          watchDebounceMs: 1500,
          intervalMinutes: 0,  // 0 = disabled
          sessions: {
            deltaBytes: 100000,
            deltaMessages: 50
          }
        },

        // Query parameters
        query: {
          maxResults: 5,
          minScore: 0.3,
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3,
            candidateMultiplier: 4
          }
        },

        // Embedding cache
        cache: {
          enabled: true,
          maxEntries: 50000
        }
      },

      // Pre-compaction memory flush
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply NO_REPLY if nothing."
        }
      }
    }
  }
}
```

---

## Integration Strategies

### Strategy A: Plugin Extension (Recommended)

Create a standalone plugin that adds external memory alongside existing system.

**Pros**:
- No core code changes
- Can coexist with builtin memory
- Full lifecycle hook access
- Easy to distribute/install

**Implementation outline**:

```typescript
// extensions/memory-hindsight/index.ts
export default {
  id: "memory-hindsight",
  kind: "memory",
  configSchema: hindsightConfigSchema,

  register(api: OpenClawPluginApi) {
    const client = new HindsightClient(api.pluginConfig);

    // Tool for explicit recall
    api.registerTool({
      name: "hindsight_search",
      description: "Search Hindsight memory for historical context",
      parameters: { query: Type.String(), limit: Type.Optional(Type.Number()) },
      async execute(_id, { query, limit = 5 }) {
        const results = await client.search(query, limit);
        return formatToolResult(results);
      }
    });

    // Auto-inject relevant context
    api.on("before_agent_start", async (event) => {
      const context = await client.getRelevantContext(event.prompt);
      if (context) {
        return { prependContext: context };
      }
    });

    // Capture important information
    api.on("agent_end", async (event) => {
      if (event.success) {
        await client.ingest(event.messages);
      }
    });

    // CLI for debugging
    api.registerCli(({ program }) => {
      const cmd = program.command("hindsight").description("Hindsight memory commands");
      cmd.command("search <query>").action(async (q) => {
        console.log(await client.search(q));
      });
      cmd.command("stats").action(async () => {
        console.log(await client.stats());
      });
    });
  }
};
```

### Strategy B: Custom Backend

Implement `MemorySearchManager` interface and register as a backend option.

**Pros**:
- Replaces builtin completely
- Uses existing tool infrastructure (`memory_search`, `memory_get`)
- Integrated status reporting

**Implementation outline**:

```typescript
// src/memory/hindsight-manager.ts
export class HindsightMemoryManager implements MemorySearchManager {
  private client: HindsightClient;

  constructor(config: HindsightConfig) {
    this.client = new HindsightClient(config);
  }

  async search(query: string, opts?: SearchOpts): Promise<MemorySearchResult[]> {
    const results = await this.client.search(query, opts?.maxResults);
    return results.map(r => ({
      path: r.source,
      startLine: r.lineStart,
      endLine: r.lineEnd,
      score: r.relevance,
      snippet: r.text,
      source: "memory"
    }));
  }

  async readFile(params: ReadParams): Promise<{ text: string; path: string }> {
    return await this.client.getFile(params.relPath, params.from, params.lines);
  }

  status(): MemoryProviderStatus {
    return {
      backend: "hindsight" as any,
      provider: "hindsight",
      model: this.client.model,
      // ... other status fields
    };
  }

  // ... implement remaining methods
}
```

Then modify `src/memory/search-manager.ts`:

```typescript
export async function getMemorySearchManager(params) {
  const resolved = resolveMemoryBackendConfig(params);

  if (resolved.backend === "hindsight") {
    const { HindsightMemoryManager } = await import("./hindsight-manager.js");
    return { manager: new HindsightMemoryManager(resolved.hindsight) };
  }

  // ... existing backend logic
}
```

### Strategy C: Parallel Memory Layer

Run external memory as a **supplementary system** that enhances the builtin.

**Approach**:
1. Keep Markdown as source of truth
2. Use `memory.qmd.paths` to index external data
3. Sync changes bidirectionally via hooks
4. External system provides enhanced retrieval

**Example flow**:
```
User message
    ↓
before_agent_start hook
    ↓
Query Hindsight API for relevant context
    ↓
Inject as prependContext
    ↓
Agent processes with enhanced context
    ↓
agent_end hook
    ↓
Push new memories to Hindsight
    ↓
Hindsight syncs to local Markdown (optional)
```

---

## File Locations

### Source Files

| Path | Description |
|------|-------------|
| `src/memory/manager.ts` | Builtin SQLite backend (2355 LOC) |
| `src/memory/qmd-manager.ts` | QMD sidecar backend (810 LOC) |
| `src/memory/search-manager.ts` | Backend factory + fallback wrapper |
| `src/memory/types.ts` | Core interfaces |
| `src/memory/embeddings.ts` | Embedding provider abstraction |
| `src/memory/embeddings-openai.ts` | OpenAI embeddings |
| `src/memory/embeddings-gemini.ts` | Gemini embeddings |
| `src/memory/embeddings-local.ts` | Local embeddings (node-llama-cpp) |
| `src/memory/hybrid.ts` | BM25 + Vector merge algorithm |
| `src/memory/manager-search.ts` | Search implementation |
| `src/memory/memory-schema.ts` | SQLite schema |
| `src/memory/batch-openai.ts` | OpenAI batch API |
| `src/memory/batch-gemini.ts` | Gemini batch API |
| `src/agents/tools/memory-tool.ts` | memory_search, memory_get tools |
| `src/agents/memory-search.ts` | Config resolution |
| `src/auto-reply/reply/memory-flush.ts` | Pre-compaction flush logic |
| `src/plugins/types.ts` | Plugin API types |
| `extensions/memory-core/` | Core memory plugin |
| `extensions/memory-lancedb/` | LanceDB plugin example |

### Runtime Locations

| Path | Content |
|------|---------|
| `~/.openclaw/config.json` | Main configuration |
| `~/.openclaw/memory/<agentId>.sqlite` | Memory index database |
| `~/.openclaw/agents/<agentId>/sessions/*.jsonl` | Session transcripts |
| `~/.openclaw/agents/<agentId>/qmd/` | QMD state directory |
| `~/clawd/` | Default workspace |
| `~/clawd/MEMORY.md` | Long-term memory |
| `~/clawd/memory/` | Daily memory logs |

---

## CLI Commands

### Memory Management

```bash
# Check memory status
openclaw memory status
openclaw memory status --deep         # Include probes
openclaw memory status --deep --index # Reindex if dirty
openclaw memory status --verbose      # Detailed logs
openclaw memory status --agent <id>   # Specific agent

# Manual indexing
openclaw memory index

# Search memories
openclaw memory search "query"
openclaw memory search "query" --limit 10
```

### Configuration

```bash
# View memory config
openclaw config get memory
openclaw config get agents.defaults.memorySearch

# Set backend
openclaw config set memory.backend qmd
openclaw config set memory.backend builtin

# Set provider
openclaw config set agents.defaults.memorySearch.provider openai
openclaw config set agents.defaults.memorySearch.provider local
```

### Debugging

```bash
# View workspace
ls ~/clawd/
cat ~/clawd/MEMORY.md
ls ~/clawd/memory/

# View memory database
sqlite3 ~/.openclaw/memory/main.sqlite ".tables"
sqlite3 ~/.openclaw/memory/main.sqlite "SELECT COUNT(*) FROM chunks"

# Check QMD state
ls ~/.openclaw/agents/main/qmd/
```

---

## Development Notes

### Testing Memory Integration

```bash
# Install and onboard
npm install -g openclaw@latest
openclaw onboard --install-daemon

# Start gateway
openclaw gateway run --verbose

# Test memory search
openclaw memory search "test query"

# Check status
openclaw memory status --deep
```

### Creating a Plugin

1. Create directory: `extensions/memory-custom/`
2. Add `package.json`:
   ```json
   {
     "name": "@openclaw/memory-custom",
     "version": "1.0.0",
     "type": "module",
     "peerDependencies": { "openclaw": ">=2026.1.26" },
     "openclaw": { "extensions": ["./index.ts"] }
   }
   ```
3. Implement plugin in `index.ts`
4. Install: `openclaw plugins install ./extensions/memory-custom`
5. Enable: `openclaw config set plugins.memory-custom.enabled true`

### Key Considerations

1. **Markdown remains source of truth** - External systems should sync, not replace
2. **Session isolation** - Memory is per-agent, use `agentId` for scoping
3. **Performance** - Embedding calls are expensive; use caching and batching
4. **Fallback** - Always have a fallback path if external service fails
5. **Privacy** - Memory contains sensitive data; secure API calls and storage
6. **Citations** - Include file paths and line numbers for transparency

### Useful Interfaces

```typescript
// Memory search result
type MemorySearchResult = {
  path: string;
  startLine: number;
  endLine: number;
  score: number;
  snippet: string;
  source: "memory" | "sessions";
  citation?: string;
};

// Plugin hook context
type PluginHookAgentContext = {
  agentId?: string;
  sessionKey?: string;
  workspaceDir?: string;
  messageProvider?: string;
};

// Before agent start result
type PluginHookBeforeAgentStartResult = {
  systemPrompt?: string;
  prependContext?: string;
};
```

---

## Next Steps for Custom Integration

1. **Evaluate requirements**:
   - Replace vs supplement existing memory?
   - Real-time vs batch ingestion?
   - Local vs cloud storage?

2. **Choose integration strategy**:
   - Plugin (fastest, most flexible)
   - Custom backend (deepest integration)
   - Parallel layer (best of both)

3. **Implement core functionality**:
   - Search/recall API
   - Ingestion/capture logic
   - Status reporting

4. **Add lifecycle hooks**:
   - `before_agent_start` for context injection
   - `agent_end` for capture
   - Optional: compaction hooks for persistence

5. **Test thoroughly**:
   - Unit tests for search accuracy
   - Integration tests with gateway
   - Performance benchmarks

6. **Document configuration**:
   - Config schema
   - Environment variables
   - CLI commands

---

*Document version: 1.0*
*Last updated: 2026-02-03*
*Based on OpenClaw v2026.2.2*
