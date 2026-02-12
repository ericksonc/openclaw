# Vector Search + BM25 Technical Overview

## Executive Summary

OpenClaw implements a **hybrid retrieval system** combining semantic vector search with BM25 keyword search. This enables agents to recall information from memory files using both conceptual understanding (vector search) and exact terminology matching (BM25).

The system is designed for:
- **Consumer users**: Auto-configured with sensible defaults
- **Power users**: Fully tunable weights, providers, and sources
- **Enterprise/privacy**: Local embeddings option for offline operation

---

## What Triggers Search

**Key insight**: Search is NOT automatic. There is no middleware that pre-fetches results or injects them into context. The agent (Claude) **autonomously decides** to search based on instructions.

### The Causality Chain

```
User sends message
       │
       ▼
System prompt is built (includes "Memory Recall" section)
       │
       ▼
Agent reads system prompt instruction:
  "Before answering anything about prior work, decisions,
   dates, people, preferences, or todos: run memory_search"
       │
       ▼
Agent reads tool description:
  "Mandatory recall step: semantically search MEMORY.md..."
       │
       ▼
Agent DECIDES whether to call memory_search
       │
       ▼
If called → search executes → results returned to agent
```

### System Prompt Instruction

From `src/agents/system-prompt.ts` (lines 40-66), the `buildMemorySection` function injects:

> "Before answering anything about prior work, decisions, dates, people, preferences, or todos: run memory_search on MEMORY.md + memory/*.md"

This section is only included if `memory_search` or `memory_get` tools are available.

### Tool Description

From `src/agents/tools/memory-tool.ts` (line 43-44):

```typescript
description: "Mandatory recall step: semantically search MEMORY.md +
  memory/*.md (and optional session transcripts) before answering
  questions about prior work, decisions, dates, people, preferences,
  or todos; returns top snippets with path + lines."
```

### What's NOT Happening

| Approach | Used? | Notes |
|----------|-------|-------|
| Automatic pre-search before turns | ❌ | No middleware |
| Auto-injection of memory into prompts | ❌ | Agent must call tool |
| Mandatory enforcement at execution time | ❌ | Agent decides |
| Hidden search based on message content | ❌ | Explicit tool call required |
| RAG-style context stuffing | ❌ | Pull model, not push |

### Why This Design?

1. **Token efficiency**: Only search when needed, not every turn
2. **Agent judgment**: Model decides if question requires recall
3. **Transparency**: Search is a visible tool call, not hidden magic
4. **Flexibility**: Agent can search multiple times or refine queries

---

## Scoping and Isolation

### What IS an Agent?

An **agent** is a **configuration profile**, not a conversation. It defines:

| Property | Purpose |
|----------|---------|
| `id` | Unique identifier (e.g., `"main"`, `"work"`, `"support"`) |
| `model` | Which LLM to use |
| `workspace` | Directory for file operations |
| `tools` | Allowlist/blocklist of capabilities |
| `memorySearch` | Search configuration (provider, sources, extraPaths) |
| `identity` | Name, avatar, emoji |
| `sandbox` | Execution isolation settings |

**Key distinction:**
- **Agent** = configuration profile (reusable)
- **Session** = active conversation (one agent, one context)
- **Relationship**: One agent → many sessions

### How Agents Are Created

Agents are **explicitly configured by users** in config files, not auto-created per conversation:

```yaml
# ~/.openclaw/config.yaml or ./openclaw.yaml
agents:
  defaults:
    model: anthropic/claude-opus-4

  list:
    - id: main
      default: true                    # Fallback when no routing matches
      workspace: ~/openclaw

    - id: work
      name: "Work Assistant"
      workspace: ~/work-projects
      tools:
        deny: [exec]                   # No shell access
      memorySearch:
        extraPaths: [~/docs/work]

    - id: support
      memorySearch:
        sources: ["memory", "sessions"] # Can search past conversations
        extraPaths: [~/docs/support-kb]
```

**If no agents configured:** A default `"main"` agent is created automatically.

### Agents vs Sessions vs Conversations

```
┌─────────────────────────────────────────────────────────────┐
│                        Agent: "work"                        │
│  (config: model, tools, workspace, memorySearch)            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Session: agent:work:telegram:dm:alice                      │
│  Session: agent:work:telegram:dm:bob                        │
│  Session: agent:work:slack:channel:eng                      │
│  Session: agent:work:whatsapp:dm:+1234567890                │
│                                                             │
│  All share the same:                                        │
│  - Tools configuration                                      │
│  - Memory database (work.sqlite)                            │
│  - Workspace directory                                      │
│                                                             │
│  Each has its own:                                          │
│  - Conversation history                                     │
│  - Session state                                            │
└─────────────────────────────────────────────────────────────┘
```

**Session key format:** `agent:{agentId}:{channel}:{context}`

Examples:
- `agent:main:telegram:dm:user123` → main agent, Telegram DM with user123
- `agent:work:slack:channel:engineering` → work agent, Slack #engineering
- `agent:support:web:session:abc123` → support agent, web chat session

### Routing Messages to Agents

Messages are routed to agents via **bindings**:

```yaml
bindings:
  - agentId: work
    match:
      channel: slack
      accountId: company-workspace    # All Slack in this workspace → work agent

  - agentId: support
    match:
      channel: telegram
      peer:
        kind: group
        id: support-channel           # This specific group → support agent

  - agentId: main                     # Fallback for everything else
```

**Resolution order:**
1. Specific peer binding (exact DM/channel match)
2. Account binding (all messages from this account)
3. Channel binding (all messages from this channel type)
4. Default agent (if nothing matches)

### One Tool, Per-Agent Databases

There is **ONE search tool** (`memory_search`) but each agent has its own **isolated SQLite database**:

```
~/.openclaw/memory/
├── agent-a.sqlite    ← Agent A's database
├── agent-b.sqlite    ← Agent B's database
└── agent-c.sqlite    ← Agent C's database
```

From `src/agents/memory-search.ts`:
```typescript
function resolveStorePath(agentId: string): string {
  return path.join(stateDir, "memory", `${agentId}.sqlite`);
}
```

**No cross-agent queries exist.** Complete isolation at the storage layer.

### Source Filtering Within a Database

Within each agent's database, content is tagged by source:

| Source | What It Contains |
|--------|------------------|
| `"memory"` | MEMORY.md, memory/*.md, extraPaths files |
| `"sessions"` | Indexed conversation transcripts (optional) |

Configuration controls which sources are searched:

```yaml
agents:
  defaults:
    memorySearch:
      sources: ["memory"]              # Default: only memory files
  list:
    - id: research-agent
      memorySearch:
        sources: ["memory", "sessions"] # This agent can search past conversations
```

Every query includes: `WHERE source IN (?, ?)` with the agent's configured sources.

### ExtraPaths: What Gets Indexed

`extraPaths` controls which files are indexed into the agent's database:

```yaml
agents:
  list:
    - id: team-agent
      memorySearch:
        extraPaths:
          - ~/docs/team-wiki      # Team docs indexed
          - ~/docs/api-reference  # API docs indexed
    - id: personal-agent
      memorySearch:
        extraPaths:
          - ~/notes/private       # Only personal notes
```

Files from different paths are all indexed with `source="memory"` but into separate per-agent databases.

### No Runtime Permissions

There is **NO ACL or permission model** within search:

| What's NOT Supported | Why |
|---------------------|-----|
| Cross-agent queries | Separate databases prevent this |
| Field-level ACLs | Not implemented |
| Role-based filtering | Not implemented |
| "Agent X can see file Y" rules | Use extraPaths instead |

**Isolation is architectural, not policy-based.** If you don't want an agent to see certain files, don't include them in that agent's `extraPaths`.

### Multi-Agent Configuration Example

```yaml
agents:
  defaults:
    memorySearch:
      provider: openai
      sources: ["memory"]
      extraPaths:
        - ~/docs/shared           # All agents see shared docs

  list:
    - id: support-agent
      memorySearch:
        sources: ["memory", "sessions"]  # Can recall past conversations
        extraPaths:
          - ~/docs/support-kb            # Support-specific knowledge

    - id: dev-agent
      memorySearch:
        provider: local                   # Privacy: local embeddings
        extraPaths:
          - ~/code/internal-docs          # Dev-specific docs

    - id: public-agent
      memorySearch:
        extraPaths: []                    # Only MEMORY.md, no extras
```

Result:
- `support-agent.sqlite` → shared + support KB + sessions
- `dev-agent.sqlite` → shared + internal docs (local embeddings)
- `public-agent.sqlite` → only MEMORY.md and memory/*.md

---

## The Problem Being Solved

### Why Search Matters for AI Agents

AI agents operate with limited context windows. They cannot keep all knowledge in memory, yet users expect them to recall:
1. Facts stored days or weeks ago
2. Decisions and rationale from past conversations
3. Personal preferences and project context

### The Dual Nature of Recall

Users search for information in two fundamentally different ways:

| Search Type | Example Query | What Works |
|-------------|--------------|------------|
| **Semantic** | "What did we decide about authentication?" | Vector search (understands concepts) |
| **Lexical** | "JWT token expiration" | BM25 (matches exact terms) |

Neither alone is sufficient. A pure vector search might miss exact terminology. A pure keyword search fails when users phrase things differently than stored text.

**Solution**: Hybrid search combines both, weighted to favor semantic (0.7) while preserving keyword precision (0.3).

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Agent Runtime                            │
├─────────────────────────────────────────────────────────────────┤
│  memory_search tool                                             │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │            SearchManager / MemoryManager                 │   │
│  │  ┌─────────────┐      ┌─────────────┐                   │   │
│  │  │   Vector    │      │    BM25     │                   │   │
│  │  │   Search    │      │   Search    │                   │   │
│  │  └──────┬──────┘      └──────┬──────┘                   │   │
│  │         │                    │                           │   │
│  │         └────────┬───────────┘                          │   │
│  │                  ▼                                       │   │
│  │           Hybrid Merger                                  │   │
│  │      (0.7 × vector + 0.3 × text)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    SQLite Database                       │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │   │
│  │  │   chunks   │  │ chunks_vec │  │    chunks_fts      │ │   │
│  │  │  (content) │  │  (vectors) │  │   (FTS5 index)     │ │   │
│  │  └────────────┘  └────────────┘  └────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Files

| File | Purpose |
|------|---------|
| `src/memory/hybrid.ts` | Hybrid merging logic, BM25 query building |
| `src/memory/manager-search.ts` | Vector & BM25 SQL queries |
| `src/memory/manager.ts` | Main memory index manager (~1000 LOC) |
| `src/memory/embeddings.ts` | Embedding provider abstraction |
| `src/memory/embeddings-openai.ts` | OpenAI embedding implementation |
| `src/memory/embeddings-gemini.ts` | Gemini embedding implementation |
| `src/memory/memory-schema.ts` | SQLite schema definitions |
| `src/agents/tools/memory-tool.ts` | Agent tool definitions |

---

## Vector Search Implementation

### Embedding Providers

Three providers with automatic fallback:

#### 1. OpenAI (Default)
```typescript
// Model: text-embedding-3-small
// Cost-optimized with batch API support
```
- **Pros**: High quality, battle-tested
- **Cons**: Requires API key, network dependency, cost

#### 2. Google Gemini
```typescript
// Model: gemini-embedding-001
// Native Gemini API integration
```
- **Pros**: Alternative cloud provider, batch support
- **Cons**: Requires API key, network dependency

#### 3. Local (node-llama-cpp)
```typescript
// Model: embeddinggemma-300M-Q8_0.gguf
// Downloaded from HuggingFace on first use
```
- **Pros**: Offline, private, no API costs
- **Cons**: Slower, requires local compute, larger memory footprint

### Auto-Selection Logic

```
provider = "auto" →
  1. Try local (if model path exists)
  2. Try OpenAI (if API key available)
  3. Try Gemini (if API key available)
  4. Use configured fallback
  5. Disable search if all fail
```

### Vector Storage

SQLite with `sqlite-vec` extension:

```sql
CREATE TABLE chunks_vec USING vec0 (
  id TEXT PRIMARY KEY,
  embedding FLOAT32[N]  -- N = vector dimensions (e.g., 1536 for OpenAI)
);
```

### Vector Query

```sql
SELECT c.id, c.path, c.text,
       vec_distance_cosine(v.embedding, ?) AS dist
  FROM chunks_vec v
  JOIN chunks c ON c.id = v.id
 WHERE c.model = ?
 ORDER BY dist ASC
 LIMIT ?
```

**Score conversion**: `score = 1 - distance` (cosine similarity, range 0-1)

---

## BM25 Keyword Search Implementation

### Full-Text Search Setup

SQLite FTS5 with BM25 ranking:

```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  id UNINDEXED,
  path UNINDEXED,
  source UNINDEXED,
  model UNINDEXED,
  start_line UNINDEXED,
  end_line UNINDEXED
);
```

### Query Tokenization

```typescript
// buildFtsQuery() in hybrid.ts
Input:  "fix authentication bug"
Tokens: ["fix", "authentication", "bug"]
Query:  '"fix" AND "authentication" AND "bug"'
```

Pattern: `/[A-Za-z0-9_]+/g` extracts alphanumeric tokens, quoted for phrase matching, joined with AND.

### BM25 Query

```sql
SELECT id, path, text, bm25(chunks_fts) AS rank
  FROM chunks_fts
 WHERE chunks_fts MATCH ?
 ORDER BY rank ASC
 LIMIT ?
```

### Score Normalization

BM25 returns negative scores (lower = better match):

```typescript
// bm25RankToScore() in hybrid.ts
score = 1 / (1 + Math.max(0, rank))

// Examples:
// rank = -5  → score = 0.167
// rank = -10 → score = 0.091
```

---

## Hybrid Merging Algorithm

### Step 1: Candidate Retrieval

```typescript
// Fetch expanded candidate pools
const vectorCandidates = maxResults * candidateMultiplier  // 6 × 4 = 24
const keywordCandidates = maxResults * candidateMultiplier // 6 × 4 = 24
// Total pool: up to 48 chunks
```

### Step 2: Score Combination

```typescript
// Merge by chunk ID
for (chunk of allCandidates) {
  finalScore = vectorWeight * vectorScore + textWeight * textScore
  //         = 0.7 × vectorScore + 0.3 × textScore (default)
}
```

### Step 3: Filtering & Ranking

```typescript
results
  .filter(r => r.score >= minScore)  // default: 0.35
  .sort((a, b) => b.score - a.score)
  .slice(0, maxResults)              // default: 6
```

### Why These Defaults?

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| `vectorWeight` | 0.7 | Semantic understanding usually more valuable |
| `textWeight` | 0.3 | Preserves keyword precision for technical terms |
| `minScore` | 0.35 | Filters noise while allowing partial matches |
| `maxResults` | 6 | Balances context quality vs. token budget |
| `candidateMultiplier` | 4 | Ensures diverse candidates before final ranking |

---

## Who Configures This and Why

### Configuration Hierarchy

```
Global defaults (built-in)
    ↓
User global config (~/.openclaw/config.yaml)
    ↓
Workspace config (./openclaw.yaml)
    ↓
Per-agent overrides (agents.{agentId}.memorySearch)
```

### Configuration Schema

```typescript
// From src/config/types.tools.ts
memorySearch: {
  enabled?: boolean;                    // Toggle search entirely

  // Provider selection
  provider?: "openai" | "gemini" | "local" | "auto";
  fallback?: "openai" | "gemini" | "local" | "none";
  model?: string;

  // What to index
  sources?: ["memory" | "sessions"];
  extraPaths?: string[];

  // Query behavior
  query?: {
    maxResults?: number;                // default: 6
    minScore?: number;                  // default: 0.35
    hybrid?: {
      enabled?: boolean;                // default: true
      vectorWeight?: number;            // default: 0.7
      textWeight?: number;              // default: 0.3
      candidateMultiplier?: number;     // default: 4
    }
  }

  // Chunking strategy
  chunking?: {
    tokens?: number;                    // default: 400
    overlap?: number;                   // default: 80
  }

  // Storage
  store?: {
    path?: string;
    vector?: {
      enabled?: boolean;
      extensionPath?: string;           // Custom sqlite-vec path
    }
  }
}
```

### Use Cases by User Type

#### Consumer (Default Config)
- Auto-selects OpenAI embeddings if API key present
- Hybrid search with 0.7/0.3 weights
- Indexes `MEMORY.md` and `memory/*.md`
- No session indexing

```yaml
# Effectively zero config needed
agents:
  defaults:
    memorySearch:
      enabled: true
```

#### Power User (Custom Weights)
- Tunes weights for domain-specific needs
- Adds extra paths (team docs, wikis)
- May prefer local embeddings for privacy

```yaml
agents:
  defaults:
    memorySearch:
      provider: local
      extraPaths:
        - ~/docs/team-wiki
      query:
        hybrid:
          vectorWeight: 0.5
          textWeight: 0.5
```

#### Enterprise (QMD Backend)
- Large knowledge bases (1000+ documents)
- Session transcript indexing
- Advanced reranking via QMD

```yaml
memory:
  backend: qmd
  qmd:
    update:
      interval: "5m"
      onBoot: true
    sessions:
      enabled: true
      retentionDays: 30
```

---

## Data Flow: Indexing

```
Memory Files                    SQLite Database
─────────────                   ───────────────
MEMORY.md        ─┐
memory/2024-01.md ├─→ Chunker ─→ chunks (content, metadata)
memory/2024-02.md ─┘      │
                          │
                          ├─→ Embedder ─→ chunks_vec (vectors)
                          │
                          └─→ FTS5 ─→ chunks_fts (tokens)
```

### Chunking Strategy

```typescript
// Default: 400 tokens per chunk, 80-token overlap
// Markdown-aware: respects paragraph/heading boundaries
// Code-aware: keeps code blocks intact when possible
```

### Sync Triggers

| Event | Action |
|-------|--------|
| Session start | Warm up embeddings |
| Search query | Sync if dirty |
| File change | Debounced reindex (1.5s) |
| Interval | Periodic reindex (optional) |

---

## Data Flow: Search

```
User Query: "authentication flow"
              │
              ▼
    ┌─────────────────────┐
    │  Parallel Queries   │
    ├──────────┬──────────┤
    │  Vector  │   BM25   │
    │  Search  │  Search  │
    └────┬─────┴────┬─────┘
         │          │
    24 results  24 results
         │          │
         └────┬─────┘
              ▼
    ┌─────────────────────┐
    │   Hybrid Merger     │
    │ 0.7×vec + 0.3×text  │
    └─────────┬───────────┘
              │
    Filter (score ≥ 0.35)
    Sort (desc by score)
    Limit (top 6)
              │
              ▼
    ┌─────────────────────┐
    │   Search Results    │
    │ [{path, lines, text,│
    │   score, snippet}]  │
    └─────────────────────┘
```

---

## Agent Integration

### memory_search Tool

```typescript
// Exposed to agents via memory-core plugin
{
  name: "memory_search",
  description: "Semantically search MEMORY.md + memory/*.md",
  parameters: {
    query: string;       // Required search query
    maxResults?: number; // Override default
    minScore?: number;   // Override default
  }
}
```

### memory_get Tool

```typescript
// Safe snippet retrieval
{
  name: "memory_get",
  description: "Read snippet from memory file",
  parameters: {
    path: string;    // Relative path
    from?: number;   // Start line
    lines?: number;  // Line count
  }
}
```

### Result Format

```typescript
{
  results: [
    {
      path: "memory/2024-01.md",
      startLine: 45,
      endLine: 52,
      text: "## Authentication\nWe decided to use JWT...",
      score: 0.82,
      snippet: "We decided to use JWT with 24h expiration..."
    }
  ],
  provider: "openai",
  model: "text-embedding-3-small",
  citations: "auto"
}
```

---

## Performance Optimizations

### Embedding Cache

```sql
CREATE TABLE embedding_cache (
  provider TEXT,
  model TEXT,
  provider_key TEXT,  -- API key fingerprint
  hash TEXT,          -- Content hash
  embedding TEXT,
  PRIMARY KEY (provider, model, provider_key, hash)
);
```

Avoids re-embedding identical content across sessions.

### Batch API

For OpenAI and Gemini, batch API reduces costs:
- Groups multiple embedding requests
- Runs asynchronously
- Polls for completion
- Configurable concurrency (default: 2)

### Query Caching

Optional in-memory cache for repeated queries:
```typescript
cache?: {
  enabled?: boolean;
  maxEntries?: number;
}
```

---

## Diagnostics

### Status Command

```bash
openclaw memory status
```

Reports:
- Backend: builtin or qmd
- Provider: embedding provider in use
- Files/Chunks: indexed content counts
- Vector extension: availability
- FTS: BM25 support status
- Fallback: if embeddings fell back

### Health Checks

```typescript
probeEmbeddingAvailability()  // Can generate embeddings?
probeVectorAvailability()     // Is sqlite-vec loaded?
```

---

## Summary

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Vector Search | sqlite-vec extension | Semantic/conceptual matching |
| BM25 Search | SQLite FTS5 | Exact keyword matching |
| Hybrid Merge | Custom algorithm | Best of both worlds |
| OpenAI Embeddings | text-embedding-3-small | Cloud semantic understanding |
| Local Embeddings | node-llama-cpp + GGUF | Offline privacy-first option |
| Storage | SQLite | Single-file portable database |

The design prioritizes:
1. **Zero-config usability** for consumers
2. **Full tunability** for power users
3. **Privacy options** via local embeddings
4. **Reliability** via fallback chains
5. **Cost efficiency** via caching and batch APIs
