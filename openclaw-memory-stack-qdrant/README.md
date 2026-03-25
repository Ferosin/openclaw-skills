# openclaw-memory-stack-qdrant

Production multi-agent memory stack for OpenClaw. Replaces the built-in QMD backend with Qdrant (vector store) + Mem0 OSS (agent-scoped memory layer) + EmbeddingGemma (local embeddings). All self-hosted, no Docker required.

## When to Use This

- Running 3+ concurrent agents (QMD has no concurrent write support)
- QMD broke after an OS reinstall or Node.js version change (native ABI mismatch)
- Need agent-scoped memory isolation (Mem0 handles this natively)
- Need cross-session fact tracking and relationship-aware recall
- Want memory that survives anything, not just compaction

For single-agent setups, the built-in QMD backend is simpler. See [openclaw-memory-resilience](../openclaw-memory-resilience/) for the compaction/file-architecture patterns that work with any backend.

## Architecture

```
OpenClaw agents
      │
      ▼
 Mem0 OSS (port 8765)          ← agent-scoped memory API
      │
 ┌────┴─────────────────┐
 │                      │
Qdrant (port 6333)   llama-server (port 8082)
Vector store         EmbeddingGemma 300M
                     Local embeddings, GPU-accelerated
```

## Stack

| Component | What It Does | Resource Cost |
|-----------|-------------|---------------|
| **Qdrant** | Stores and retrieves vectors. Concurrent HTTP API, no locking. | ~50MB RAM |
| **EmbeddingGemma 300M** | Converts text to 768-dim vectors. Fully local. | ~300MB RAM + GPU |
| **Mem0 OSS** | Agent_id scoping, fact extraction, cross-session memory. | ~100MB RAM |
| **Small reasoning model** | Extracts structured facts from raw text. | Local or cloud API |

Total overhead on a system with ≥16GB RAM: trivial. EmbeddingGemma runs on GPU (Metal/CUDA) so CPU impact is near-zero.

## Why This Stack

**vs QMD:** QMD is an embedded binary with native Node.js ABI dependency. Any OS/runtime change can break it with no graceful fallback. Qdrant is a standalone Rust binary — it doesn't care about your Node.js version. See [decision-matrix](./references/decision-matrix.md).

**vs ChromaDB:** ChromaDB has known "Database is locked" errors under concurrent writes. Not suitable for multi-agent setups.

**vs pgvector:** Good alternative if you're already on Postgres. Qdrant is simpler if you just need a vector store.

**vs hosted solutions (Pinecone, etc.):** Your memory stays on your hardware. No API costs, no data leaving your network.

## Quick Start

1. Install and start Qdrant (native binary, LaunchAgent)
2. Download EmbeddingGemma GGUF, start via llama-server (LaunchAgent)
3. Install Mem0 in a Python venv, start server (LaunchAgent)
4. Build the `memory-mem0` OpenClaw plugin
5. Index existing memory files
6. Disable QMD cron, create incremental reindex cron

Full walkthrough in [SKILL.md](./SKILL.md).

## Ports

| Service | Port | Notes |
|---------|------|-------|
| Qdrant REST | 6333 | Primary API |
| Qdrant gRPC | 6334 | Optional |
| llama-server (embed) | 8082 | EmbeddingGemma only |
| Mem0 | 8765 | Memory API |

No conflicts with standard services.

## References

- [Decision matrix](./references/decision-matrix.md) — full backend comparison
- [Troubleshooting](./references/troubleshooting.md) — common failure modes and fixes

## Installation

```bash
clawhub install openclaw-memory-stack-qdrant
```

Or manually copy this directory into `<your-workspace>/skills/`.

## License

MIT
