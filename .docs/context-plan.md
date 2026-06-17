# Plan: Check and Configure NVIDIA Nemotron-3-Ultra Context Limits

## Objective
Determine the actual context window of `nvidia/nemotron-3-ultra-550b-a55b` via NVIDIA NIM API and configure `danno.toml` appropriately for larger context.

## Current State
- Using `models.nemotron-ultra` (NVIDIA NIM endpoint: `https://integrate.api.nvidia.com/v1`)
- `context_budget = 128000` in `[backends.nvidia]` — this is only OpenCode's *client-side belief*
- No `num_ctx` parameter is sent to NIM (OpenAI-compatible API ignores it per Ollama behavior, but NIM may differ)

## Steps

### 1. Check NVIDIA NIM Model Limits
```bash
# Query the model info endpoint (if available)
curl -H "Authorization: Bearer $NVIDIA_API_KEY" \
  https://integrate.api.nvidia.com/v1/models/nvidia/nemotron-3-ultra-550b-a55b

# Or try a test completion with large context to see actual limit
curl -X POST https://integrate.api.nvidia.com/v1/chat/completions \
  -H "Authorization: Bearer $NVIDIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/nemotron-3-ultra-550b-a55b",
    "messages": [{"role": "user", "content": "test"}],
    "max_tokens": 100
  }'
```

### 2. Check NVIDIA Documentation
- Visit: https://build.nvidia.com/nvidia/nemotron-3-ultra
- Look for "Context Length" or "Max Tokens" in model card
- Check NIM deployment docs for `max_model_len` or similar parameter

### 3. Test Actual Context via API
```bash
# Create a test with progressively larger inputs until error
# This determines the practical limit
```

### 4. Configure danno.toml Based on Findings

**If NIM supports `max_tokens`/`num_ctx` parameter:**
```toml
[backends.nvidia]
kind        = "openai"
base_url    = "https://integrate.api.nvidia.com/v1"
api_key_env = "NVIDIA_API_KEY"
context_budget = <ACTUAL_LIMIT>   # e.g., 128000, 256000, 1000000
output_limit   = 8192
```

**If NIM has fixed context (no parameter control):**
- Keep `context_budget` at the documented limit
- OpenCode will trim/compact to this budget client-side

### 5. Apply and Rebuild
```bash
cd /Users/mikestitt/projects/first/2027/romi-wpilib-photonvision
uv run danno install --apply
uv run danno sandbox rebuild --apply
uv run danno sandbox start
```

## Questions for User
1. Do you have `NVIDIA_API_KEY` set in your shell for testing?
2. Should I also check if a local Ollama model (gemma3:27b / qwen3.6:27b with custom Modelfile) would be better for 256k context?
3. Target context size: 128k, 256k, or 1M+?

## Notes
- NIM may have different limits than self-hosted Nemotron
- The `context_budget` in danno.toml only affects OpenCode's conversation trimming, not the API call
- For true context increase, the API must accept and honor a context parameter