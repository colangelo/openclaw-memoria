# External Memory Systems: Deep Comparison

> Analysis of external memory systems (Hindsight, Graphiti, Mem0, Zep, Letta) vs local approaches (CoreMemories, NATS Event Sourcing) for OpenClaw integration.
> Last updated: 2026-02-03

---

## Executive Summary

| System | Philosophy | Best For |
|--------|------------|----------|
| **CoreMemories** | Human-like decay | Token reduction, single user, offline |
| **NATS JetStream** | Event sourcing | Audit trails, time travel, multi-agent |
| **Mem0** | Adaptive synthesis | Multi-user, proven accuracy, SaaS-friendly |
| **Zep/Graphiti** | Temporal knowledge graph | Relationships, change tracking, enterprise |
| **Letta** | Self-improving agents | Structured memory, agent autonomy |
| **Hindsight** | Biomimetic learning | Learning + reflection, SOTA retrieval |

**Key Insight:** CoreMemories solves "how do I reduce tokens?" The external systems solve "how do I make AI that actually learns?"

---

## NATS JetStream Event Sourcing

> PR #7358 - Production-validated (157K+ events, 30 days uptime)

### What Is Event Sourcing?

**Traditional Memory:**
```
User says something → Extract facts → Store in DB → Query later
                            ↓
                    (original context lost)
```

**Event Sourcing:**
```
User says something → Store RAW EVENT → Build ANY view later
                            ↓
                    (nothing lost, ever)
```

### Killer Features

#### 1. Time Travel Queries
```
"What did we decide about the API last Tuesday?"
"What was my preference BEFORE I changed it?"
"Show me the context from when we started this project"
```

**Why this matters:** CoreMemories compresses Flash → Warm, losing original detail forever. With event sourcing, you can reconstruct any point in time.

#### 2. Full State Reconstruction
```typescript
// Lost context? Replay from any point
await replayEvents({ since: "2024-01-01", agent: "main" });

// Debugging? See exactly what happened
await replayEvents({ agent: "main", verbose: true });

// Audit trail? It's automatic
await getEvents({ agent: "main", type: "decision" });
```

#### 3. Multiple Views from Same Data

Same events can simultaneously power:
- Learning system (extract preferences)
- Analytics dashboard (usage patterns)
- Knowledge graph (entity relationships)
- Compliance report (audit trail)
- Fact extractor (structured data)

CoreMemories has ONE fixed view (Flash/Warm/Archive). Event sourcing has unlimited views.

#### 4. Multi-Agent Coordination

Production stats from PR author:
| Metric | Value |
|--------|-------|
| Events Processed | 157,000+ |
| Agents Coordinated | 5 |
| Continuous Uptime | 30 days |
| Event Publish Latency | <5ms |

Each agent has private stream + can subscribe to others' events.

### When to Choose NATS

**Choose NATS if you need:**
- Audit trails or compliance
- Questions about history ("what changed?")
- Multiple coordinated agents
- Flexibility to build new views without re-ingesting
- Debug/replay capabilities

**Don't choose NATS if:**
- You just need "what's relevant now" (still need retrieval layer)
- Infrastructure complexity is a concern
- Single-user personal assistant use case

### NATS + External Memory

NATS is an **event backbone**, not a retrieval system. Best architecture:

```
User interaction
       ↓
   NATS JetStream (stores raw events)
       ↓
   ┌───┴───┐
   ↓       ↓
Hindsight  Graphiti  (build views from events)
   ↓       ↓
Retrieval layer
```

---

## CoreMemories (PR #7480)

> Local-first hierarchical memory inspired by human cognition

### Architecture

```
HOT LAYER (Always loaded, ~1400 tokens)
├── Flash (0-48h): Full conversation detail
└── Warm (2-7d): Summaries + key quotes

RECENT LAYER (Triggered by keyword match)
└── Week 1-4: Progressive compression (50% → 20% detail)

ARCHIVE LAYER (Deep retrieval only)
└── Fresh → Core (1 month → 1+ years): Essence extraction
```

### Strengths

| Aspect | Benefit |
|--------|---------|
| Local-first | Uses Ollama, no API costs |
| Token-efficient | 30-40% reduction in memory tokens |
| Human-like | Recent = vivid, old = faded |
| Simple | Easy mental model, easy to debug |
| Private | Never leaves your machine |

### Weaknesses

| Aspect | Limitation |
|--------|------------|
| Lossy compression | Once Flash → Warm, original detail gone forever |
| Keyword retrieval | No semantic search for older memories |
| No relationships | Doesn't understand entity connections |
| No temporal queries | Can't ask "what changed between X and Y" |
| Single-user | No multi-agent or multi-user support |
| No learning | Compresses, doesn't synthesize new knowledge |

### When to Choose CoreMemories

**Choose if:**
- Zero cloud dependencies required
- Token cost is primary concern
- Simple personal assistant use case
- You already run Ollama
- Privacy is paramount

**Don't choose if:**
- You need relationship understanding
- You need temporal queries
- You want the agent to learn/improve
- Multi-user or multi-agent scenarios

---

## Mem0

> Multi-level adaptive memory with active synthesis

### Architecture

```
┌─────────────────────────────────────┐
│         User Memories               │  ← Long-term preferences
│    (persistent across sessions)     │
├─────────────────────────────────────┤
│        Session Memories             │  ← Conversation context
│     (within specific interaction)   │
├─────────────────────────────────────┤
│         Agent Memories              │  ← Behavioral patterns
│    (autonomous system learning)     │
└─────────────────────────────────────┘
```

### Key Differentiator: Synthesis

CoreMemories **compresses** text:
```
"User said they like Python" + "User asked about types"
→ Compressed: "User discussed Python and types"
```

Mem0 **synthesizes** knowledge:
```
"User said they like Python" + "User asked about types"
→ Synthesized: "User is interested in typed Python (likely mypy/Pydantic)"
```

### Benchmark Performance

| Metric | Mem0 | Comparison |
|--------|------|------------|
| Accuracy | 26% higher | vs OpenAI Memory |
| Speed | 91% faster | vs full-context approaches |
| Token usage | 90% lower | vs full-context approaches |

### Comparison with CoreMemories

| Aspect | CoreMemories | Mem0 |
|--------|--------------|------|
| Extraction | Rule-based compression | LLM-powered synthesis |
| Multi-user | No | Yes (user/session/agent) |
| Accuracy | Unknown | 26% better (benchmarked) |
| Token efficiency | 30-40% reduction | 90% reduction |
| Retrieval speed | Local only | 91% faster |
| Learning | Compression only | Creates new knowledge |
| Deployment | Self-hosted only | Cloud or self-hosted |

### When to Choose Mem0

**Choose if:**
- Multi-user support needed
- You want proven benchmarks
- SaaS deployment acceptable
- User/session/agent isolation required
- You need active knowledge synthesis

**Don't choose if:**
- Fully offline requirement
- Don't trust cloud services
- Simple single-user case

---

## Zep + Graphiti

> Temporal knowledge graph with relationship awareness

### Architecture

```
Events/Messages
       ↓
   Graphiti extracts
       ↓
┌──────────────────────────────────────────┐
│           Knowledge Graph                 │
│                                          │
│  [Sam]──works_at──▶[Acme]                │
│    │                  │                  │
│    │ valid_at: 2024-01  invalid_at: 2025-06
│    │                                     │
│    └──works_at──▶[StartupX]              │
│         valid_at: 2025-06                │
│                                          │
└──────────────────────────────────────────┘
```

### Key Differentiator: Bi-Temporal Tracking

Every fact has TWO timestamps:
1. **Event time**: When it actually occurred
2. **Ingestion time**: When it entered the system

This enables:
```
Q: "What was Sam's role in January 2025?"
A: "Engineer at Acme"

Q: "What's Sam's role now?"
A: "CTO at StartupX (since June 2025)"

Q: "When did Sam's role change?"
A: "June 2025 - left Acme, joined StartupX as CTO"
```

CoreMemories would just say "Sam is CTO" and lose the history.

### Contradiction Handling

| System | Contradiction Handling |
|--------|----------------------|
| CoreMemories | LLM summarizes away (lossy) |
| Graphiti | Edge invalidation (preserved) |

```
Old fact: "Sam works at Acme"
New fact: "Sam works at StartupX"

CoreMemories: Overwrites, old fact gone
Graphiti: Marks old edge invalid_at=now, creates new edge
```

### Retrieval: Hybrid Approach

Graphiti combines:
1. **Semantic embeddings** (meaning similarity)
2. **BM25 keyword search** (exact matches)
3. **Graph traversal** (relationship paths)

All with sub-second latency.

### Comparison with CoreMemories

| Aspect | CoreMemories | Zep/Graphiti |
|--------|--------------|--------------|
| Relationships | None | Full graph with traversal |
| Time tracking | Age-based decay | Bi-temporal (event + ingestion) |
| Query types | Keywords | Graph + semantic + BM25 |
| Change tracking | Overwritten | valid_at/invalid_at preserved |
| Contradictions | Summarized away | Edge invalidation |
| Scale | Single user | Enterprise-grade |

### When to Choose Zep/Graphiti

**Choose if:**
- Relationships between entities matter
- Temporal queries needed ("what changed?")
- Enterprise/compliance requirements
- You're building knowledge graphs
- Change tracking is important

**Don't choose if:**
- Simple Q&A without relationships
- Offline-only requirement
- No need for historical queries

---

## Letta (formerly MemGPT)

> Agents that self-improve through structured memory blocks

### Architecture

```
┌─────────────────────────────────────┐
│         Memory Blocks               │
│   (structured, agent-editable)      │
├─────────────────────────────────────┤
│  human: {                           │
│    name: "Alice",                   │
│    preferences: ["Python", "CLI"],  │
│    context: "Working on API"        │
│  }                                  │
├─────────────────────────────────────┤
│  persona: {                         │
│    identity: "Helpful assistant",   │
│    behavior: "Concise responses"    │
│  }                                  │
├─────────────────────────────────────┤
│  custom: {                          │
│    project_state: {...},            │
│    learned_patterns: [...]          │
│  }                                  │
└─────────────────────────────────────┘
```

### Key Differentiator: Agent Self-Management

| System | Memory Control |
|--------|----------------|
| CoreMemories | Passive storage (agent reads/writes files) |
| Letta | Agent actively manages its own memory blocks |

The agent can:
- Edit its own persona
- Update its understanding of the user
- Create new memory blocks as needed
- Delete outdated information

This enables **self-improvement loops**:
```
Interaction → Learn → Update memory → Better next interaction
```

### Comparison with CoreMemories

| Aspect | CoreMemories | Letta |
|--------|--------------|-------|
| Structure | Flat markdown chunks | Structured typed blocks |
| Agent control | Passive read/write | Active self-management |
| Learning | Compression only | Self-improvement loops |
| Context control | Time-based (Flash/Warm) | Block-based (you choose) |
| Memory editing | File operations | First-class block operations |

### When to Choose Letta

**Choose if:**
- You want agents that self-improve
- Structured memory blocks needed (not just text)
- Building autonomous agents
- Agent should control its own memory
- You want the MemGPT paradigm

**Don't choose if:**
- Simple assistant without autonomy
- Unstructured memory is fine
- Don't need agent self-management

---

## Hindsight

> Biomimetic memory with learning, not just remembering

### Architecture: Three Pathways

```
┌─────────────────────────────────────────────────────┐
│                    HINDSIGHT                         │
├─────────────────┬─────────────────┬─────────────────┤
│     WORLD       │   EXPERIENCES   │  MENTAL MODELS  │
│                 │                 │                 │
│ Factual         │ Agent's         │ Learned         │
│ knowledge       │ interactions    │ understanding   │
│                 │                 │                 │
│ "What IS"       │ "What HAPPENED" │ "What it MEANS" │
└─────────────────┴─────────────────┴─────────────────┘
```

### Three Core Operations

#### 1. Retain (Store)
```
Input: Raw text/conversation
   ↓
LLM extracts:
├── Facts
├── Entities
├── Relationships
└── Temporal data
   ↓
Normalized into searchable indexes
```

#### 2. Recall (Retrieve)
```
Query
   ↓
Four parallel strategies:
├── Semantic vector similarity
├── BM25 keyword matching
├── Entity/temporal graph connections
└── Time range filtering
   ↓
Reciprocal rank fusion
   ↓
Cross-encoder reranking
   ↓
Results
```

#### 3. Reflect (Learn) ← **Unique to Hindsight**
```
Existing memories
   ↓
Analysis for:
├── Pattern recognition
├── Risk assessment
├── New connections
└── Insight generation
   ↓
NEW knowledge (not just retrieval)
```

### Key Differentiator: Reflect Operation

Other systems store and retrieve. Hindsight **thinks**.

```
Memories:
- "User asked about Python testing 5 times"
- "User mentioned pytest twice"
- "User complained about slow tests once"

Reflect output:
- "User is focused on Python testing"
- "User prefers pytest framework"
- "User may benefit from test optimization tips"
- "Pattern: Testing questions cluster on Mondays"
```

This creates **new knowledge** that was never explicitly stated.

### Comparison with CoreMemories

| Aspect | CoreMemories | Hindsight |
|--------|--------------|-----------|
| Retrieval | Keywords only | 4 parallel strategies + reranking |
| Learning | None | Reflect creates new knowledge |
| Structure | Time-based layers | Semantic (World/Experience/Mental) |
| Relationships | None | Entity-relationship graph |
| Multi-user | No | Metadata-based isolation |
| Benchmark | Unknown | SOTA on LongMemEval |
| Production | PR stage | Fortune 500 deployed |

### Retrieval Quality

Hindsight's 4-strategy + reranking approach:

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Semantic   │  │    BM25     │  │   Graph     │  │    Time     │
│   Vector    │  │   Keyword   │  │  Traversal  │  │   Range     │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       └────────────────┴────────────────┴────────────────┘
                               │
                    Reciprocal Rank Fusion
                               │
                    Cross-encoder Reranking
                               │
                         Final Results
```

CoreMemories: Keywords only, no fusion, no reranking.

### When to Choose Hindsight

**Choose if:**
- You want **learning**, not just remembering
- Reflect operation valuable (insight generation)
- Need SOTA retrieval quality
- Building at Fortune 500 scale
- Want biomimetic memory (World/Experience/Mental)

**Don't choose if:**
- Simple storage/retrieval sufficient
- Can't use external service
- Cost is primary concern

---

## Decision Matrix

### By Use Case

| Use Case | Recommended | Why |
|----------|-------------|-----|
| Personal assistant (single user) | CoreMemories | Simple, local, free |
| Multi-user SaaS | Mem0 | Built for multi-tenancy |
| Enterprise with compliance | Zep/Graphiti | Audit trails, temporal tracking |
| Autonomous agents | Letta | Self-improvement capability |
| Learning AI | Hindsight | Reflect operation |
| Multi-agent coordination | NATS + any | Event backbone |
| Relationship-heavy domain | Graphiti | Knowledge graph |
| Audit/debug requirements | NATS | Full event replay |

### By Constraint

| Constraint | Recommended | Avoid |
|------------|-------------|-------|
| Must be offline | CoreMemories | Mem0, Zep, Hindsight |
| Zero cost | CoreMemories + Ollama | Cloud services |
| Proven accuracy | Mem0, Hindsight | Unvalidated |
| Need time travel | NATS, Graphiti | CoreMemories |
| Need relationships | Graphiti, Hindsight | CoreMemories |
| Need learning | Hindsight, Letta | CoreMemories |

### By Complexity Budget

| Complexity | System | Infrastructure |
|------------|--------|----------------|
| Minimal | CoreMemories | Ollama only |
| Low | Mem0 Cloud | API key only |
| Medium | Hindsight | API + config |
| High | Graphiti | Neo4j + embeddings |
| Highest | NATS + Graphiti | Full event infra |

---

## Recommended Architecture for OpenClaw

### Option 1: Simple (CoreMemories-like)
```
OpenClaw → Local memory files → Ollama compression
```
Best for: Personal use, privacy-focused, cost-conscious

### Option 2: Learning (Hindsight)
```
OpenClaw → Hindsight API
              ├── Retain (on message)
              ├── Recall (before response)
              └── Reflect (periodic)
```
Best for: AI that improves over time, insight generation

### Option 3: Relationships (Graphiti)
```
OpenClaw → Graphiti
              ├── Entity extraction
              ├── Relationship tracking
              └── Temporal queries
```
Best for: Complex domains, enterprise, compliance

### Option 4: Full Stack (NATS + Hindsight/Graphiti)
```
OpenClaw → NATS JetStream (event backbone)
              ├── → Hindsight (learning layer)
              ├── → Graphiti (relationship layer)
              └── → Analytics (usage patterns)
```
Best for: Maximum capability, multi-agent, enterprise

---

## Integration Priority Recommendation

For OpenClaw, based on community gaps and capability analysis:

### Tier 1: High Value, No Existing Solution
1. **Hindsight** - Reflect operation is unique, SOTA retrieval
2. **Graphiti** - Powers Zep, best temporal knowledge graph

### Tier 2: Valuable, Some Overlap
3. **Mem0** - Good benchmarks, but overlaps with memory-redis PR
4. **NATS** - Already has PR #7358, validate and merge

### Tier 3: Specialized
5. **Letta** - Only if building autonomous agents

---

## Links & References

| System | Repository | Docs |
|--------|------------|------|
| Mem0 | [github.com/mem0ai/mem0](https://github.com/mem0ai/mem0) | [docs.mem0.ai](https://docs.mem0.ai) |
| Zep | [github.com/getzep/zep](https://github.com/getzep/zep) | [docs.getzep.com](https://docs.getzep.com) |
| Graphiti | [github.com/getzep/graphiti](https://github.com/getzep/graphiti) | In repo |
| Letta | [github.com/letta-ai/letta](https://github.com/letta-ai/letta) | [docs.letta.com](https://docs.letta.com) |
| Hindsight | [github.com/vectorize-io/hindsight](https://github.com/vectorize-io/hindsight) | In repo |

---

*Document version: 1.0*
*Last updated: 2026-02-03*
