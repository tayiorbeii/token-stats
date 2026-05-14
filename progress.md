# Research Progress

## 2026-05-14: External Cache Token Handling Research

### Status: COMPLETE ✅

### Task
Research how other token-stats tools handle Cache Create tokens for OpenAI GPT models (o3, o4-mini, gpt-4.1, etc.) in the context of Codex and Pi agent usage.

### What was done
1. Searched **ccusage** (ryoppippi/ccusage) for cache token handling — found complete OpenAI and Anthropic code paths
2. Searched **splitrail** (Piebald-AI/splitrail) — found the cleanest caching abstraction with 3-strategy model
3. Searched **toktrack** and **openlit** for additional cache token handling patterns
4. Documented pricing multipliers for all major OpenAI and Anthropic models

### Key Findings
- **OpenAI uses single `cached_input_tokens`** (cache read only, 50-90% discount)
- **Anthropic uses two fields**: `cache_creation_input_tokens` (1.25× input, write) + `cache_read_input_tokens` (0.10× input, read)
- **ccusage subtracts cached from input** for OpenAI; uses **additive model** for Anthropic
- **splitrail ignores cache_creation_tokens entirely for OpenAI** — only cache_read_tokens is billed
- **Newer OpenAI models (o3, o4-mini, gpt-4.1) use 75% cache discount**; GPT-5 family uses 90%

### Output
- `/Users/taylor/Documents/Projects/03-tools/token-stats/research-findings-external.md` — Full research brief
