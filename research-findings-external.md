# Research: External Token-Stats Tools — Cache Token Handling for OpenAI GPT Models

## Summary

External tools handle cache tokens fundamentally differently between OpenAI and Anthropic providers. **OpenAI uses a single `cached_input_tokens` field** (cache read only, 50-90% discount from input price), while **Anthropic uses two separate fields** (`cache_creation_input_tokens` at 1.25× input price, and `cache_read_input_tokens` at 0.10× input price). For OpenAI models, most tools subtract cached tokens from the input count before applying regular input pricing, then apply the discounted cached rate to the cached portion separately. No tool treats OpenAI `cache_creation` as a separate billable category — OpenAI only exposes cache read, not cache write.

---

## Findings

### 1. OpenAI Cache Token Model — Single Rate, Subset of Input

**OpenAI exposes only `cached_input_tokens` (a.k.a. `cache_read_input_tokens`)** — there is no separate `cache_creation_input_tokens` field in OpenAI API responses.

From **ccusage** (`apps/codex/src/data-loader.ts`):
```typescript
// Accepts both field names as aliases
const cached = ensureNumber(record.cached_input_tokens ?? record.cache_read_input_tokens);
```

From **ccusage** (`apps/codex/CLAUDE.md` — billing treatment documentation):

| Field | Meaning | Billing treatment |
|---|---|---|
| `input_tokens` | Prompt tokens sent this turn | Priced at `input_cost_per_mtoken` minus the cached share |
| `cached_input_tokens`/`cache_read_input_tokens` | Prompt tokens satisfied from cache | Priced at `cached_input_cost_per_mtoken` (falls back to input price) |
| `output_tokens` | Completion tokens, including reasoning cost | Priced at `output_cost_per_mtoken` |

**Source**: [ccusage](https://github.com/ryoppippi/ccusage) — `apps/codex/src/data-loader.ts`, `apps/codex/CLAUDE.md`

---

### 2. ccusage Cost Calculation — OpenAI (Codex) Path

ccusage treats `cachedInputTokens` as a **subset of `inputTokens`** and subtracts it to avoid double-charging.

From **ccusage** (`apps/codex/src/token-utils.ts`):
```typescript
function nonCachedInputTokens(usage: TokenUsageDelta): number {
    const nonCached = usage.inputTokens - usage.cachedInputTokens;
    return nonCached > 0 ? nonCached : 0;
}

export function calculateCostUSD(usage: TokenUsageDelta, pricing: ModelPricing): number {
    const nonCachedInput = nonCachedInputTokens(usage);
    const cachedInput = usage.cachedInputTokens > usage.inputTokens
        ? usage.inputTokens
        : usage.cachedInputTokens;

    const inputCost  = (nonCachedInput / MILLION) * pricing.inputCostPerMToken;
    const cachedCost = (cachedInput / MILLION) * pricing.cachedInputCostPerMToken;
    const outputCost = (outputTokens / MILLION) * pricing.outputCostPerMToken;

    return inputCost + cachedCost + outputCost;
}
```

**Formula**: `cost = (input - cached) × input_rate + cached × cached_rate + output × output_rate`

**Source**: [ccusage](https://github.com/ryoppippi/ccusage) — `apps/codex/src/token-utils.ts`

---

### 3. ccusage Cost Calculation — Anthropic (Claude) Path

For Anthropic, ccusage uses an **additive model** — `input_tokens` is the FULL count and cache tokens are billed as **separate additional line items**.

From **ccusage** (`packages/internal/src/pricing.ts`):
```typescript
const inputCost = calculateTieredCost(input_tokens, input_cost_per_token, above_200k_price);
const outputCost = calculateTieredCost(output_tokens, output_cost_per_token, above_200k_price);
const cacheCreationCost = calculateTieredCost(cache_creation_tokens, cache_creation_price, above_200k_price);
const cacheReadCost = calculateTieredCost(cache_read_tokens, cache_read_price, above_200k_price);

return inputCost + outputCost + cacheCreationCost + cacheReadCost;
```

**Formula**: `cost = input × input_rate + output × output_rate + creation × write_rate + read × read_rate`

Note the tiered pricing support with `above_200k_tokens` thresholds:
```typescript
cache_creation_input_token_cost_above_200k_tokens: v.optional(v.number()),
cache_read_input_token_cost_above_200k_tokens: v.optional(v.number()),
```

**Source**: [ccusage](https://github.com/ryoppippi/ccusage) — `packages/internal/src/pricing.ts`

---

### 4. splitrail's Three-Strategy Caching Model

**splitrail** (Piebald-AI/splitrail, Rust) explicitly models three different caching strategies:

```rust
pub enum CachingSupport {
    None,
    OpenAI   { cached_input_per_1m: f64 },              // Single cached rate
    Anthropic{ cache_write_per_1m: f64, cache_read_per_1m: f64 },  // Two rates
    Google(TieredCaching),                               // Tiered by context length
}
```

Cost calculation dispatches by strategy:

```rust
fn cache_cost_for_model(model_info: &ModelInfo, cache_creation_tokens: u64, cache_read_tokens: u64) -> f64 {
    match &model_info.caching {
        CachingSupport::None => 0.0,

        CachingSupport::OpenAI { cached_input_per_1m } => {
            // OpenAI: ONLY uses cache_read_tokens, ignores cache_creation_tokens
            (cache_read_tokens as f64 / 1_000_000.0) * cached_input_per_1m
        }

        CachingSupport::Anthropic { cache_write_per_1m, cache_read_per_1m } => {
            let creation_cost = (cache_creation_tokens as f64 / 1_000_000.0) * cache_write_per_1m;
            let read_cost     = (cache_read_tokens as f64 / 1_000_000.0) * cache_read_per_1m;
            creation_cost + read_cost
        }

        CachingSupport::Google(tiered) => {
            calculate_tiered_cache_cost(cache_read_tokens, &tiered.tiers, tiered.bracket_pricing)
        }
    }
}
```

**Critical insight**: For OpenAI, `cache_creation_tokens` is **ignored entirely** — only `cache_read_tokens` matters.

**Source**: [splitrail](https://github.com/Piebald-AI/splitrail) — `src/models.rs`

---

### 5. OpenAI Model Pricing (Cached Rates)

From **splitrail** (`src/models.rs`) — hardcoded pricing for OpenAI models:

| Model | Input/MTok | Output/MTok | Cached/MTok | Cache Discount |
|---|---|---|---|---|
| **o4-mini** | $1.10 | $4.40 | $0.275 | **75%** |
| **o3** | $2.00 | $8.00 | $0.50 | **75%** |
| **o3-mini** | $1.10 | $4.40 | $0.55 | 50% |
| **gpt-4.1** | $2.00 | $8.00 | $0.50 | **75%** |
| **gpt-4.1-mini** | $0.40 | $1.60 | $0.10 | **75%** |
| **gpt-4.1-nano** | $0.10 | $0.40 | $0.025 | **75%** |
| **gpt-5** | $1.25 | $10.00 | $0.125 | **90%** |
| **gpt-5-mini** | $0.25 | $2.00 | $0.025 | **90%** |
| **gpt-5-nano** | $0.05 | $0.40 | $0.005 | **90%** |
| o1 | $15.00 | $60.00 | $7.50 | 50% |
| o1-mini | $1.10 | $4.40 | $0.55 | 50% |
| gpt-4o | $2.50 | $10.00 | $1.25 | 50% |
| gpt-4o-mini | $0.15 | $0.60 | $0.075 | 50% |

**Key pattern**: Newer OpenAI models (o3, o4-mini, gpt-4.1 family) use 75% discount for cached tokens. GPT-5 family uses 90% discount. Legacy models (o1, gpt-4o) use 50%.

**Source**: [splitrail](https://github.com/Piebald-AI/splitrail) — `src/models.rs`

---

### 6. Anthropic Cache Token Pricing (For Comparison)

From **splitrail** (`src/models.rs`) — Anthropic models:

| Model | Input/MTok | Output/MTok | Cache Write/MTok | Cache Read/MTok | Write Mult | Read Mult |
|---|---|---|---|---|---|---|
| claude-opus-4-7 | $5.00 | $25.00 | $6.25 | $0.50 | 1.25× | 0.10× |
| claude-sonnet-4 | $3.00 | $15.00 | $3.75 | $0.30 | 1.25× | 0.10× |
| claude-3-5-sonnet | $3.00 | $15.00 | $3.75 | $0.30 | 1.25× | 0.10× |
| claude-3-5-haiku | $0.80 | $4.00 | $1.00 | $0.08 | 1.25× | 0.10× |
| claude-haiku-4-5 | $1.00 | $5.00 | $1.25 | $0.10 | 1.25× | 0.10× |
| claude-3-haiku | $0.25 | $1.25 | $0.30 | $0.03 | 1.20× | 0.12× |

**Formula** (all Anthropic models except claude-3-haiku):
- `cache_write = input_price × 1.25` (25% premium)
- `cache_read = input_price × 0.10` (90% discount)

**Source**: [splitrail](https://github.com/Piebald-AI/splitrail) — `src/models.rs`

---

### 7. ccusage's LiteLLM-Based Pricing Fetcher

ccusage fetches pricing dynamically from LiteLLM's model pricing JSON rather than hardcoding. The pricing schema from `packages/internal/src/pricing.ts`:

```typescript
export const liteLLMModelPricingSchema = v.object({
  input_cost_per_token: v.optional(v.number()),
  output_cost_per_token: v.optional(v.number()),
  cache_creation_input_token_cost: v.optional(v.number()),
  cache_read_input_token_cost: v.optional(v.number()),
  // Tiered pricing above 200k tokens:
  input_cost_per_token_above_200k_tokens: v.optional(v.number()),
  output_cost_per_token_above_200k_tokens: v.optional(v.number()),
  cache_creation_input_token_cost_above_200k_tokens: v.optional(v.number()),
  cache_read_input_token_cost_above_200k_tokens: v.optional(v.number()),
  // Also 128k tier:
  input_cost_per_token_above_128k_tokens: v.optional(v.number()),
  output_cost_per_token_above_128k_tokens: v.optional(v.number()),
  // Speed multiplier:
  provider_specific_entry: v.optional(v.object({ fast: v.optional(v.number()) })),
});
```

**Example test data** (Claude 4 Sonnet):
```typescript
'anthropic/claude-4-sonnet-20250514': {
  input_cost_per_token: 3e-6,           // $3/MTok
  output_cost_per_token: 1.5e-5,        // $15/MTok
  cache_creation_input_token_cost: 3.75e-6,  // $3.75/MTok
  cache_read_input_token_cost: 3e-7,    // $0.30/MTok
  cache_creation_input_token_cost_above_200k_tokens: 7.5e-6,
  cache_read_input_token_cost_above_200k_tokens: 6e-7,
}
```

**Source**: [ccusage](https://github.com/ryoppippi/ccusage) — `packages/internal/src/pricing.ts`

---

### 8. How Other Tools Map Provider-Specific Fields

#### ccusage — Pi Agent (`apps/pi/src/_pi-agent.ts`):
```typescript
cache_creation_input_tokens: usage.cacheWrite ?? 0,
cache_read_input_tokens: usage.cacheRead ?? 0,
```

#### ccusage — OpenCode (`apps/opencode/src/cost-utils.ts`):
```typescript
input_tokens: entry.usage.inputTokens,
output_tokens: entry.usage.outputTokens,
cache_creation_input_tokens: entry.usage.cacheCreationInputTokens,
cache_read_input_tokens: entry.usage.cacheReadInputTokens,
```

#### ccusage — Amp (`apps/amp/src/pricing.ts`):
```typescript
input_tokens: tokens.inputTokens,
output_tokens: tokens.outputTokens,
cache_creation_input_tokens: tokens.cacheCreationInputTokens,
cache_read_input_tokens: tokens.cacheReadInputTokens,
```

#### ccusage — Codex (`apps/codex/src/data-loader.ts`):
```typescript
// Dual-key support (alias handling)
const cached = ensureNumber(record.cached_input_tokens ?? record.cache_read_input_tokens);
```

#### toktrack — Claude (`src/parsers/claude.rs`):
```rust
assert_eq!(first.cache_creation_tokens, 10);
assert_eq!(first.cache_read_tokens, 20);
```

**Source**: [ccusage](https://github.com/ryoppippi/ccusage), [toktrack](https://github.com/mag123c/toktrack)

---

### 9. Display Handling — Do Tools Separate Cache Tokens in Output?

From **ccusage** (`apps/ccusage/src/commands/_session_id.ts`):
```typescript
// Separate columns in session display
formatNumber(entry.message.usage.cache_creation_input_tokens ?? 0),
formatNumber(entry.message.usage.cache_read_input_tokens ?? 0),
formatCurrency(entry.costUSD ?? 0),
```

ccusage displays cache_creation and cache_read as **separate columns** in session output, alongside regular input/output token counts.

**splitrail** also separates cache costs in its display, using the three-strategy model to show the relevant cache metrics per provider.

**Source**: [ccusage](https://github.com/ryoppippi/ccusage) — `apps/ccusage/src/commands/_session_id.ts`

---

## Sources

### Kept
- **[ccusage](https://github.com/ryoppippi/ccusage)** — Most comprehensive TypeScript token tracker. Multi-provider with separate OpenAI/Anthropic code paths. Core pricing module with LiteLLM integration, tiered pricing, and speed multiplier support. [File: `packages/internal/src/pricing.ts`, `apps/codex/src/token-utils.ts`, `apps/codex/src/data-loader.ts`]
- **[splitrail](https://github.com/Piebald-AI/splitrail)** — Rust-based tracker with the cleanest caching abstraction: `CachingSupport` enum with `None`, `OpenAI`, `Anthropic`, `Google` variants. Hardcoded pricing tables for all major models. [File: `src/models.rs`]
- **[toktrack](https://github.com/mag123c/toktrack)** — Rust-based ultra-fast tracker. Handles cache_creation and cache_read tokens from Claude Code JSONL. [File: `src/parsers/claude.rs`]
- **[openlit](https://github.com/openlit/openlit)** — Full observability SDK (Python/Go/TypeScript). Auto-instruments Anthropic/OpenAI with cache token tracking via OpenTelemetry spans. [Files: `sdk/go/instrumentation/anthropic/types.go`, `sdk/typescript/src/semantic-convention.ts`]

### Dropped
- **token-tracker (stormzhang)** — Python tool with custom StatusLine, but cache handling appears less sophisticated than ccusage.
- **cccost (badlogic)** — TypeScript tool that instruments Claude Code, but focused on Claude only, not multi-provider.
- **claude-code-monitor (zcquant)** — JavaScript OTLP exporter, no local pricing/cost calculation.

---

## Gaps

1. **OpenAI `cache_creation_input_tokens` handling**: No tool was found that exposes or bills for OpenAI `cache_creation_input_tokens` as a separate category. OpenAI's API appears to only report cached (read) tokens, not cache write tokens. The Anthropic path is the only one with a cache_creation (write) billing component.

2. **Pi Agent specific cache fields**: ccusage maps Pi's `cacheWrite` and `cacheRead` to Anthropic-style `cache_creation_input_tokens` and `cache_read_input_tokens`, but the underlying Pi data source may use Anthropic or OpenAI models depending on configuration. The pricing treatment may not match the actual provider.

3. **Codex + Pi agent exact pricing for OpenAI models**: ccusage's Codex path correctly subtracts cached from input for OpenAI, but the Pi agent path uses the Anthropic additive model. If Pi uses OpenAI models (o3, o4-mini), this could result in incorrect cost calculations.

4. **Live Anthropic/OpenAI pricing page verification**: Could not directly fetch the Anthropic or OpenAI pricing documentation pages to verify the hardcoded prices in splitrail. The pricing tables should be cross-referenced with official docs.

5. **Reasoning token pricing**: Some OpenAI models (o3, o4-mini) have separate `reasoning_output_tokens` fields — not all tools handle these distinctly from regular output tokens in their pricing.
