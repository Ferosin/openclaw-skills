# Memory Backend Decision Matrix

## Why Not Just Use QMD?

QMD is OpenClaw's built-in memory backend -- hybrid BM25 + vector search, single embedded binary. It's excellent for single-agent setups and requires zero additional infrastructure.

Two problems arise at scale:

1. **Fragile binary:** QMD uses `better-sqlite3`, a native Node.js module compiled against a specific ABI. Any OS reinstall, Node.js version change, or Bun/Node mismatch can produce `ERR_DLOPEN_FAILED` and leave all agents running blind. There's no graceful degradation.

2. **No concurrent write support:** QMD is an embedded database (SQLite-backed). Multiple agents writing simultaneously can hit locking issues. Not a problem with 1-2 agents, but breaks at 3+.

## Candidate Comparison

| Criterion | QMD | Qdrant | Mem0+Qdrant | ChromaDB | pgvector |
|-----------|-----|--------|-------------|----------|----------|
| Multi-agent concurrent writes | No | Yes | Yes | No | Yes |
| Agent/session scoping | No | Manual | Native | No | Manual |
| Cross-session fact tracking | No | Manual | Native | No | Manual |
| Hybrid search (dense + BM25) | Yes | Yes (v1.7+) | Yes | No | Needs extension |
| macOS ARM native binary | Fragile | Yes | Yes | Yes | Yes |
| Runtime stability | Fragile | Very stable | Very stable | Moderate | Very stable |
| Self-hosted | Yes | Yes | Yes | Yes | Yes |
| Setup complexity | Was minimal | Low | Moderate | Low | Moderate |
| Docker required | No | No | No | Optional | No |
| Concurrent write safety | No | Yes | Yes | No | Yes (MVCC) |

## Why Not ChromaDB?

ChromaDB shows "Database is locked" errors under concurrent multi-agent writes. This is a known limitation documented in Mem0's own production guides. Not recommended for any multi-agent setup.

## Why Not pgvector?

pgvector is a solid choice if you're already running PostgreSQL (e.g., for Forgejo or another service). It handles concurrent access via MVCC and supports SQL filtering.

The downside: no native BM25 hybrid search without the ParadeDB/pg_search extension. And it adds operational complexity if you're not already managing a Postgres instance.

**Use pgvector if:** You're already on Postgres and want everything in one database.
**Use Qdrant if:** You want a purpose-built vector store with minimal ops overhead.

## Why Qdrant?

- Rust-built: very stable, low memory overhead
- Native macOS ARM binary (no Rosetta, no Docker)
- Concurrent HTTP API: no locking
- Hybrid search (dense + sparse BM25) native since v1.7
- Payload filtering for agent/project/date scoping
- Active development: $50M Series B (March 2026)

## Why Mem0 on Top of Qdrant?

Qdrant is a vector store -- it stores and retrieves vectors. It doesn't know about agents, sessions, or facts. You'd have to implement all of that yourself.

Mem0 adds:
- Native `agent_id` and `user_id` scoping (no schema design required)
- Fact extraction: converts raw text into structured memories via LLM
- Cross-session persistence: memories accumulate across conversations
- REST API: compatible with OpenClaw's plugin slot system

The combination is the canonical self-hosted production stack for multi-agent memory.
