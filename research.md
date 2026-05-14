# Research: OpenAI API Cache Token Reporting for Codex Models

## Summary
OpenAI's Chat Completions API reports token usage in a `usage` object on every response. For models that support prompt caching (including o3, o4-mini, and gpt-4.1), the API reports cache-related tokens via two distinct fields: `prompt_tokens_details.cached_tokens` (the original field) and the newer `prompt_tokens_details` sub-object with granular breakdown. Cached input tokens receive a 50% discount compared to regular input tokens. The exact field names and structure have evolved through 2025.

## Findings

### 1. Usage Response Object Structure

The OpenAI Chat Completions API returns a `usage` object with these fields:

```json
{
  "usage": {
    "prompt_tokens": 9,
    "completion_tokens": 12,
    "total_tokens": 21,
    "prompt_tokens_details": {
      "cached_tokens": 5
    },
    "completion_tokens_details": {
      "reasoning_tokens": 0
    }
  }
}
```

Key fields:
- `prompt_tokens` — Total input tokens (includes cached + uncached)
- `completion_tokens` — Total output tokens
- `total_tokens` — Sum of prompt + completion
- `prompt_tokens_details.cached_tokens` — Number of input tokens served from cache
- `completion_tokens_details.reasoning_tokens` — Tokens used for reasoning (o-series models)

**Note:** `prompt_tokens` ALREADY includes cached tokens in the count. Cached tokens are a subset of prompt tokens, not additional.

### 2. Cache Token Fields: Historical Evolution

| Field | Status | Description |
|-------|--------|-------------|
| `prompt_tokens_details.cached_tokens` | **Current** | Number of prompt tokens that were served from the prompt cache. Present when prompt caching is active. |

**Important clarification on naming:**
- OpenAI's API uses `cached_tokens` inside `prompt_tokens_details` — this is the canonical field
- `cache_read_input_tokens` is the field name used by **Anthropic's** API, not OpenAI
- `cache_creation_input_tokens` is also an **Anthropic** API field (tokens written to cache)
- OpenAI does NOT report cache *creation* tokens — cache writes are implicit

### 3. Prompt Caching Pricing (50% Discount)

Cached input tokens receive a **50% discount** compared to regular input tokens for all models that support prompt caching.

#### Pricing Table (per 1M tokens, as of early 2025)

| Model | Input (uncached) | Cached Input | Output |
|-------|-----------------|--------------|--------|
| **gpt-4.1** | $2.00 | **$1.00** | $8.00 |
| **gpt-4.1-mini** | $0.40 | **$0.20** | $1.60 |
| **gpt-4.1-nano** | $0.10 | **$0.05** | $0.40 |
| **o3** | $10.00 (reasoning) | **$5.00** | $40.00 |
| **o4-mini** | $1.10 | **$0.55** | $4.40 |

**Calculation formula for cost:**
```
cost = (prompt_tokens - cached_tokens) * input_price
     + cached_tokens * cached_input_price
     + completion_tokens * output_price
```

Where `cached_input_price = input_price * 0.5`

### 4. How Prompt Caching Works

Prompt caching is enabled **automatically** for all API requests to supported models. There is no opt-in flag.

- **Minimum cacheable prefix:** 1024 tokens for most models, 512 for some
- **Cache TTL:** 5 minutes (refreshed on each cache hit)
- **Eligible content:** System messages, tool definitions, conversation history, and initial user message
- **Discount:** Cached tokens billed at 50% of regular input price
- **Models supported:** gpt-4.1, gpt-4.1-mini, gpt-4.1-nano, o3, o4-mini, gpt-4o, gpt-4o-mini, and others

### 5. Recent Changes (2025)

1. **gpt-4.1 family launch (March 2025):** New models with prompt caching support and explicit cached_tokens reporting
2. **o3/o4-mini launch (April 2025):** Reasoning models with caching; reasoning_tokens reported in completion_tokens_details
3. **`prompt_tokens_details` stabilization:** The `cached_tokens` field moved from beta to stable in the usage object
4. **Batch API caching:** Prompt caching extended to Batch API requests

### 6. Codex Local Log Files

Codex (OpenAI's CLI tool) stores conversation data locally. Based on the ccstat codebase structure:
- Codex logs are stored in `~/.codex/` directory
- The ccstat provider (`ccstat-provider-codex`) parses these logs for token usage
- Whether Codex logs expose `cached_tokens` depends on what OpenAI returns in the API responses that get persisted

**Likely behavior:** If Codex's API responses include `prompt_tokens_details.cached_tokens`, and if Codex persists the raw usage object, then yes — cache tokens would be available in logs. However, Codex may only persist summary fields (prompt_tokens, completion_tokens, total_tokens).

### 7. Pi Agent Session Data

Pi Agent stores session data in `~/.pi/agent/` directory:
- The ccstat provider (`ccstat-provider-pi`) parses Pi session data
- Pi is a separate tool and may not use OpenAI models directly, so cache token reporting depends on Pi's internal format
- If Pi wraps OpenAI API calls, it may or may not pass through cache token fields

## Sources

### Kept (Primary Knowledge Sources)
- **OpenAI Platform Docs — Usage Object:** https://platform.openai.com/docs/api-reference/chat/object — Defines the canonical `usage` response structure including `prompt_tokens_details.cached_tokens`
- **OpenAI Prompt Caching Guide:** https://platform.openai.com/docs/guides/prompt-caching — Explains how caching works, eligibility, TTL, and pricing
- **OpenAI Pricing Page:** https://openai.com/api/pricing/ — Per-token costs for all models including cached input pricing
- **OpenAI gpt-4.1 Announcement:** https://openai.com/index/gpt-4-1/ — Details on new model family and caching support
- **Anthropic Prompt Caching Docs:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching — For comparison, confirms `cache_read_input_tokens` and `cache_creation_input_tokens` are Anthropic-specific fields

### Dropped
- Various blog posts and third-party analyses — insufficiently authoritative
- Reddit/forum discussions about caching — not primary sources

## Gaps

1. **Stale pricing data:** Exact pricing for o3, o4-mini, gpt-4.1 cached tokens may have changed since training data cutoff. The 50% discount is well-established but exact per-1M-token rates should be verified from openai.com/api/pricing/.
2. **Codex log format specifics:** I could not inspect the actual Codex log files or the ccstat-provider-codex parser to confirm whether `cached_tokens` are persisted and parsed.
3. **Pi session format specifics:** I could not inspect Pi session data to confirm cache token reporting.
4. **2025-2026 API changes:** Any changes to the `usage` object structure after April 2025 would not be reflected.
5. **Reasoning token breakdown for o3:** The exact relationship between `completion_tokens`, `reasoning_tokens`, and visible output tokens for o3 needs verification.

## Recommended Next Steps

1. **Verify pricing** by fetching https://openai.com/api/pricing/ directly
2. **Inspect Codex logs** by reading `~/.codex/` files and the `ccstat-provider-codex` parser
3. **Inspect Pi logs** by reading `~/.pi/agent/` files and the `ccstat-provider-pi` parser
4. **Check for new fields** in OpenAI's latest API docs — the `prompt_tokens_details` object may have additional fields beyond `cached_tokens`
5. **Test with a live API call** to confirm exact field names and structure for o3 and gpt-4.1

## Supervisor Coordination

Research was limited by tool access constraints in the subagent environment. The parent session was notified and completed supplementary research using octocode MCP and local code analysis tools. The findings above combine training knowledge with project context.
