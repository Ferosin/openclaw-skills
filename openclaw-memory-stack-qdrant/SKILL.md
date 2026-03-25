---
name: openclaw-memory-stack-qdrant
description: >
  Set up a production multi-agent memory stack for OpenClaw using Qdrant (vector store),
  Mem0 OSS (agent-scoped memory layer), EmbeddingGemma (local embeddings), and a small
  reasoning model (fact extraction). Use when: running 3+ concurrent agents, needing
  agent-scoped memory isolation, QMD broke after an OS/Node.js version change, or needing
  cross-session recall and relationship tracking. All components self-hosted, no cloud
  dependency for memory operations, no Docker required.
---

# OpenClaw Memory Stack: Qdrant + Mem0

Production-grade multi-agent memory stack. All components run as native LaunchAgents
(macOS) or systemd services (Linux). No Docker required.

## When to Use This vs QMD

| Situation | Use |
|-----------|-----|
| Single agent, simple setup | QMD (built into OpenClaw) |
| QMD broke after OS/Node.js update | This stack |
| 3+ concurrent agents | This stack |
| Need agent-scoped memory isolation | This stack |
| Need cross-session fact tracking | This stack |

See `references/decision-matrix.md` for the full comparison.

## Architecture

```
OpenClaw agents
      |
      v
 Mem0 OSS API          ← agent-scoped memory layer (port 8765)
      |
 ┌────┴─────────────────┐
 |                      |
Qdrant               llama-server
(vector store)       (embeddings, GPU-accelerated)
port 6333            port 8082
                         |
                  EmbeddingGemma GGUF
                  (300M params, ~300MB RAM)
```

**How memory flows:**
1. Agent calls `memory_search` → OpenClaw memory-mem0 plugin → Mem0 REST API
2. Mem0 embeds the query via EmbeddingGemma (local, no cloud)
3. Mem0 searches Qdrant for semantically similar vectors
4. Results return to agent with agent_id scoping applied

**On memory writes:**
1. Agent calls memory write → Mem0 receives text
2. Small reasoning model extracts structured facts (configurable: local or cloud)
3. Facts stored as vectors in Qdrant, scoped to agent_id

## Components

| Component | Role | Port | Notes |
|-----------|------|------|-------|
| Qdrant | Vector store | 6333 (HTTP), 6334 (gRPC) | Native binary, persistent storage |
| EmbeddingGemma | Embeddings | 8082 | Via llama-server, full GPU acceleration |
| Mem0 OSS | Memory layer | 8765 | Agent_id scoping, fact extraction |
| reasoning model | Fact extraction | — | Local (Qwen/Llama) or cloud API |

## Phase 1: Qdrant

**Download and install:**
```bash
# macOS ARM64
curl -L https://github.com/qdrant/qdrant/releases/latest/download/qdrant-aarch64-apple-darwin.tar.gz \
  -o /tmp/qdrant.tar.gz
tar -xzf /tmp/qdrant.tar.gz -C /opt/homebrew/bin/
chmod +x /opt/homebrew/bin/qdrant

# Linux ARM64: use qdrant-aarch64-unknown-linux-musl.tar.gz
# Linux x86: use qdrant-x86_64-unknown-linux-musl.tar.gz
```

**Config** (`/path/to/storage/qdrant/config.yaml`):
```yaml
storage:
  storage_path: /path/to/storage/qdrant/storage
  snapshots_path: /path/to/storage/qdrant/snapshots

service:
  host: 0.0.0.0
  http_port: 6333
  grpc_port: 6334
```

**macOS LaunchAgent** (`~/Library/LaunchAgents/com.yourname.qdrant.plist`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.yourname.qdrant</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/qdrant</string>
    <string>--config-path</string>
    <string>/path/to/storage/qdrant/config.yaml</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key>
  <string>/tmp/qdrant.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/qdrant.error.log</string>
</dict>
</plist>
```

**Verify:**
```bash
launchctl load ~/Library/LaunchAgents/com.yourname.qdrant.plist
curl http://localhost:6333/
# Should return Qdrant version info JSON
```

## Phase 2: EmbeddingGemma via llama-server

EmbeddingGemma (300M) is Google's purpose-built embedding model. At 300M params it uses ~300MB RAM and runs fully on GPU via Metal (macOS) or CUDA (Linux).

**Install llama.cpp (if not already installed):**
```bash
brew install llama.cpp    # macOS
# Linux: build from source or use official binaries from github.com/ggerganov/llama.cpp
```

**Download the GGUF model:**

Check `ggml-org/embeddinggemma-300m-*` on HuggingFace for the current release.
Verify the exact filename before downloading -- the Q8_0 variant (~300MB) is recommended.

```bash
# Example (verify filename against current HF release):
huggingface-cli download ggml-org/embeddinggemma-300m-Q8_0-GGUF \
  embeddinggemma-300m-q8_0.gguf \
  --local-dir /path/to/models/
```

**macOS LaunchAgent:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.yourname.llama-embed</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/llama-server</string>
    <string>-m</string>
    <string>/path/to/models/embeddinggemma-300m-q8_0.gguf</string>
    <string>--embeddings</string>
    <string>--pooling</string><string>mean</string>
    <string>-ngl</string><string>999</string>
    <string>--port</string><string>8082</string>
    <string>--host</string><string>0.0.0.0</string>
    <string>-c</string><string>2048</string>
    <string>--no-webui</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key>
  <string>/tmp/llama-embed.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/llama-embed.error.log</string>
</dict>
</plist>
```

**Key flags:**
- `--embeddings` — enables the embeddings endpoint
- `--pooling mean` — mean pooling for sentence-level vectors (required for EmbeddingGemma)
- `-ngl 999` — offload all layers to GPU (Metal on macOS, CUDA on Linux)

**Verify:**
```bash
curl http://localhost:8082/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"embeddinggemma","input":"test sentence"}'
# Should return a JSON object with a "data" array containing a vector
```

## Phase 3: Mem0 OSS

Mem0 is the agent-aware layer that sits in front of Qdrant. It handles agent_id scoping, fact extraction, and the REST API that OpenClaw's plugin calls.

**Install:**
```bash
# Create a dedicated venv (recommended — Mem0 has specific dependency pinning)
python3 -m venv ~/venvs/mem0
source ~/venvs/mem0/bin/activate
pip install "mem0ai[server]"
```

**Config** (`/path/to/mem0/config.yaml`):
```yaml
vector_store:
  provider: qdrant
  config:
    host: localhost
    port: 6333
    collection_name: agent_memory

embedder:
  provider: openai          # llama-server is OpenAI-compatible
  config:
    model: embeddinggemma
    openai_base_url: http://localhost:8082/v1
    api_key: "dummy"        # llama-server requires the field but ignores the value

llm:
  provider: openai          # Can be anthropic, ollama, or any OpenAI-compatible endpoint
  config:
    model: gpt-4o-mini      # Small fast model for fact extraction only
    # Or use a local model:
    # openai_base_url: http://localhost:8083/v1
    # model: qwen3.5-4b
    api_key: "${OPENAI_API_KEY}"
```

**Reasoning model options for fact extraction:**
- Cloud: `claude-haiku-4-5` (Anthropic), `gpt-4o-mini` (OpenAI) — low cost, extraction calls only
- Local: Any small model (Qwen 3.5B, Llama 3.2 3B) via llama-server or Ollama on a separate port
- Local is preferred if you want zero cloud dependency for memory operations

**macOS LaunchAgent:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.yourname.mem0</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/youruser/venvs/mem0/bin/python</string>
    <string>-m</string>
    <string>mem0.server</string>
    <string>--port</string>
    <string>8765</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>MEM0_CONFIG</key>
    <string>/path/to/mem0/config.yaml</string>
    <key>OPENAI_API_KEY</key>
    <string>your-key-here</string>
  </dict>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key>
  <string>/tmp/mem0.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/mem0.error.log</string>
</dict>
</plist>
```

**Verify:**
```bash
launchctl load ~/Library/LaunchAgents/com.yourname.mem0.plist

# Test add
curl -X POST http://localhost:8765/v1/memories/ \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "My name is Alex"}], "agent_id": "myagent"}'

# Test search
curl -X POST http://localhost:8765/v1/memories/search/ \
  -H "Content-Type: application/json" \
  -d '{"query": "name", "agent_id": "myagent"}'
```

## Phase 4: OpenClaw Plugin

OpenClaw's memory schema only supports `builtin` and `qmd` as named backends. Mem0 integrates via the **plugin slot system** (`plugins.slots.memory`).

Build a `memory-mem0` plugin in your workspace:

```
workspace/plugins/memory-mem0/
├── openclaw.plugin.json
├── package.json
└── index.ts
```

`openclaw.plugin.json`:
```json
{
  "id": "memory-mem0",
  "name": "Mem0 Memory Backend",
  "version": "1.0.0",
  "slots": ["memory"]
}
```

The plugin implements `memory_search` and `memory_get`, proxying to Mem0's REST API. Reference the bundled `memory-lancedb` plugin in your OpenClaw install for the exact interface signatures.

Apply via `openclaw gateway config.patch`:
```json
{
  "plugins": {
    "slots": { "memory": "memory-mem0" },
    "entries": {
      "memory-core": { "enabled": false },
      "memory-lancedb": { "enabled": false }
    }
  }
}
```

## Phase 5: Index Existing Memory Files

```python
#!/usr/bin/env python3
# index-memory.py — indexes workspace memory files into Qdrant via Mem0
import os, glob
from mem0 import Memory

m = Memory()

# Map agent_id → memory directory
agents = {
    "shared": "/path/to/workspace/memory/",
    "agent1": "/path/to/workspace-agent1/memory/",
    # add more agents here
}

for agent_id, base_dir in agents.items():
    if not os.path.exists(base_dir):
        print(f"Skipping {agent_id}: directory not found")
        continue
    files = glob.glob(f"{base_dir}**/*.md", recursive=True)
    print(f"Indexing {len(files)} files for agent: {agent_id}")
    for filepath in files:
        try:
            content = open(filepath, encoding="utf-8").read()
            if len(content.strip()) < 50:
                continue
            m.add(content, agent_id=agent_id, metadata={"source": filepath})
        except Exception as e:
            print(f"  Error {filepath}: {e}")

print("Done.")
```

For nightly incremental indexing (only files modified in the last 24h), filter by `os.path.getmtime` before the `m.add` call.

## Verification Checklist

```bash
# Qdrant running
curl http://localhost:6333/

# Embeddings working
curl http://localhost:8082/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"embeddinggemma","input":"test"}'

# Mem0 running
curl http://localhost:8765/

# End-to-end: add and search
curl -X POST http://localhost:8765/v1/memories/ \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"The API key is stored in the config file"}],"agent_id":"test"}'

curl -X POST http://localhost:8765/v1/memories/search/ \
  -H "Content-Type: application/json" \
  -d '{"query":"where is the API key","agent_id":"test"}'
```

## References

- `references/decision-matrix.md` — QMD vs Qdrant vs Mem0+Qdrant vs ChromaDB vs pgvector
- `references/ports.md` — port map and conflict avoidance
- `references/troubleshooting.md` — common failure modes and fixes
