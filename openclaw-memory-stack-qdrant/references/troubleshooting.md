# Troubleshooting

## Qdrant

**Qdrant won't start:**
```bash
tail -f /tmp/qdrant.error.log
# Common: storage directory doesn't exist or isn't writable
mkdir -p /path/to/storage/qdrant/{storage,snapshots}
```

**External drive dependency:** If Qdrant storage is on a removable/external drive, the drive must be mounted before Qdrant starts. Add a `StartInterval` delay to the LaunchAgent or use a dependency plist if needed.

**Port conflict:**
```bash
lsof -i :6333
# Kill the conflicting process or change Qdrant's port in config.yaml
```

## EmbeddingGemma / llama-server

**No vectors returned (empty `data` array):**
- Confirm `--embeddings` flag is present in the LaunchAgent
- Confirm `--pooling mean` is set (required for EmbeddingGemma)
- Check logs: `tail -f /tmp/llama-embed.error.log`

**Model not loading:**
- Verify the GGUF file path is correct and the file isn't corrupted
- Try: `llama-server -m /path/to/model.gguf --embeddings --pooling mean --port 8082`

**GPU not being used (slow performance):**
- Confirm `-ngl 999` is set (offloads all layers)
- macOS: check Metal is available with `system_profiler SPDisplaysDataType`
- Linux: confirm CUDA is installed and `llama-server` was built with CUDA support

## Mem0

**`mem0.server` module not found:**
```bash
# Verify installation
source ~/venvs/mem0/bin/activate
python -c "import mem0; print(mem0.__version__)"
# Check current server invocation in Mem0 docs -- module path may differ by version
```

**Mem0 connects to Qdrant but embeddings fail:**
- Confirm llama-server is running on port 8082: `curl http://localhost:8082/v1/embeddings -d '{"model":"embeddinggemma","input":"test"}'`
- Check `openai_base_url` in Mem0 config points to the correct port
- The `api_key: "dummy"` field is required even though llama-server ignores the value

**Fact extraction fails (LLM errors):**
- Check API key is set correctly in the LaunchAgent `EnvironmentVariables`
- If using a local reasoning model, verify its server is running on the correct port
- Reduce extraction model to a cheaper/smaller option if hitting rate limits

**Memory search returns nothing after indexing:**
- Verify the collection name in Mem0 config matches what was created in Qdrant
- Check Qdrant dashboard: `http://localhost:6333/dashboard` — confirm vectors exist
- Re-run the index script with verbose logging

## OpenClaw Plugin

**`memory_search` still using QMD after plugin install:**
- Confirm plugin is in `plugins.slots.memory` in gateway config
- Confirm `memory-core` and `memory-lancedb` are disabled
- Restart gateway: `openclaw gateway restart`
- Run `/context list` to check which memory plugin is active

**Plugin not loading:**
- Check `openclaw.plugin.json` has `"slots": ["memory"]`
- Check plugin directory is in the `plugins.entries` or `plugins.paths` config
- Check gateway logs: `openclaw gateway logs`
