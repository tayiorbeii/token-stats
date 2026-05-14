# Research Findings: Cache Token Handling in ccstat (Codex & Pi Providers)

## Files Retrieved

1. `crates/ccstat-core/src/types.rs` (lines 185-550) - Core token types: `TokenCounts`, `MessageUsage`, `ModelPricing`, `UsageEntry`
2. `crates/ccstat-core/src/provider.rs` (full, 24 lines) - `ProviderDataLoader` trait definition
3. `crates/ccstat-core/src/aggregation_types.rs` (scanned) - All aggregation structs use `TokenCounts`
4. `crates/ccstat-provider-codex/src/data_loader.rs` (full, 438 lines) - Codex JSONL parser and delta computation
5. `crates/ccstat-provider-pi/src/data_loader.rs` (full, 378 lines) - Pi JSONL parser
6. `crates/ccstat-pricing/src/cost_calculator.rs` (full, 347 lines) - Cost calculation with cache token support
7. `crates/ccstat-pricing/src/pricing_fetcher.rs` (full, 485 lines) - LiteLLM pricing data fetcher
8. `crates/ccstat-terminal/src/output.rs` (scanned for cache references) - Display layer fully supports cache tokens
9. `crates/ccstat-pricing/embedded/pricing.json` (scanned for OpenAI/Claude models) - Pricing data availability

---

## Key Code

### Core Types (`crates/ccstat-core/src/types.rs`)

**`TokenCounts`** (lines 201-210): The universal token struct used across all providers:
```rust
pub struct TokenCounts {
    pub input_tokens: u64,
    pub output_tokens: u64,
    pub cache_creation_tokens: u64,    // Cache creation tokens
    pub cache_read_tokens: u64,        // Cache read tokens
}
```

**`MessageUsage`** (lines 347-359): Raw API usage from Claude JSONL:
```rust
pub struct MessageUsage {
    pub input_tokens: u64,
    #[serde(default)]
    pub output_tokens: u64,
    #[serde(default)]
    pub cache_creation_input_tokens: u64,
    #[serde(default)]
    pub cache_read_input_tokens: u64,
}
```

**`ModelPricing`** (lines 334-343): Pricing with cache support:
```rust
pub struct ModelPricing {
    pub input_cost_per_token: Option<f64>,
    pub output_cost_per_token: Option<f64>,
    pub cache_creation_input_token_cost: Option<f64>,
    pub cache_read_input_token_cost: Option<f64>,
}
```

**`UsageEntry::from_raw`** (lines 534-538): Claude provider correctly maps all 4 token fields:
```rust
tokens: TokenCounts {
    input_tokens: raw.message.usage.input_tokens,
    output_tokens: raw.message.usage.output_tokens,
    cache_creation_tokens: raw.message.usage.cache_creation_input_tokens,
    cache_read_tokens: raw.message.usage.cache_read_input_tokens,
},
```

### Codex Provider (`crates/ccstat-provider-codex/src/data_loader.rs`)

**`CumulativeTokens`** (lines 123-137): Raw Codex JSONL token structure:
```rust
struct CumulativeTokens {
    #[serde(default)]
    input_tokens: u64,
    #[serde(default)]
    cached_input_tokens: u64,
    #[serde(alias = "cache_read_input_tokens")]
    #[serde(default)]
    cache_read_tokens: u64,
    #[serde(default)]
    output_tokens: u64,
    #[serde(default)]
    total_tokens: u64,
}
```

**`compute_delta`** (lines 243-283): Converts cumulative → per-event delta:
```rust
fn compute_delta(info: &TokenInfo, prev: Option<&CumulativeTokens>) -> TokenCounts {
    if let Some(last) = &info.last_token_usage {
        let cache_read = if last.cached_input_tokens > 0 {
            last.cached_input_tokens
        } else {
            last.cache_read_tokens
        };
        return TokenCounts::new(last.input_tokens, last.output_tokens, 0, cache_read);
    }
    // ... cumulative delta path also handles cache_read
    TokenCounts::new(
        total.input_tokens.saturating_sub(prev_input),
        total.output_tokens.saturating_sub(prev_output),
        0,  // ⚠️ cache_creation is ALWAYS 0
        cache_read_total.saturating_sub(prev_cache_read),
    )
}
```

**Zero-token skip** (line 221): Only checks input and output:
```rust
if delta.input_tokens == 0 && delta.output_tokens == 0 {
    continue;
}
```

### Pi Provider (`crates/ccstat-provider-pi/src/data_loader.rs`)

**`PiUsage`** (lines 120-133): Pi JSONL usage structure:
```rust
#[serde(rename_all = "camelCase")]
struct PiUsage {
    #[serde(default)]
    input: u64,
    #[serde(default)]
    output: u64,
    #[serde(default)]
    cache_read: u64,
    #[serde(default)]
    cache_write: u64,
    #[serde(default)]
    cost: Option<PiCost>,
}
```

**Token mapping** (lines 211-216): Pi correctly maps all 4 fields:
```rust
tokens: TokenCounts::new(
    usage.input,
    usage.output,
    usage.cache_write,   // → cache_creation_tokens
    usage.cache_read,    // → cache_read_tokens
),
```

**Zero-token skip** (line 184): Only checks input and output:
```rust
if usage.input == 0 && usage.output == 0 {
    continue;
}
```

### Pricing (`crates/ccstat-pricing/src/cost_calculator.rs`)

**`calculate_from_pricing`** (lines 106-132): All 4 token types are priced:
```rust
if let Some(rate) = pricing.input_cost_per_token {
    cost += tokens.input_tokens as f64 * rate;
}
if let Some(rate) = pricing.output_cost_per_token {
    cost += tokens.output_tokens as f64 * rate;
}
if let Some(rate) = pricing.cache_creation_input_token_cost {
    cost += tokens.cache_creation_tokens as f64 * rate;
}
if let Some(rate) = pricing.cache_read_input_token_cost {
    cost += tokens.cache_read_tokens as f64 * rate;
}
```

### Pricing Data (`crates/ccstat-pricing/embedded/pricing.json`)

**OpenAI models (used by Codex)**: Most GPT-5.x models include `cache_read_input_token_cost`:
- `openai/gpt-5`: `cache_read_input_token_cost: 1.25e-07` ✅
- `openai/gpt-5-codex`: `cache_read_input_token_cost: 1.25e-07` ✅
- `openai/gpt-5-mini`: `cache_read_input_token_cost: 2.5e-08` ✅
- `openai/gpt-5.1-codex`: `cache_read_input_token_cost: 1.25e-07` ✅
- `openai/gpt-5.2-codex`: `cache_read_input_token_cost: 1.75e-07` ✅
- **NONE** have `cache_creation_input_token_cost` — OpenAI doesn't expose this concept.

**Claude models (used by Pi)**: Include both cache fields:
- `claude-sonnet-4-5-20250929`: `cache_read_input_token_cost: 0.0000003` ✅
- Claude models also have `cache_creation_input_token_cost` ✅

---

## Architecture

```
Log Files (JSONL)
  ├── Claude: ~/.claude/projects/**/*.jsonl → RawJsonlEntry → UsageEntry (all 4 token fields)
  ├── Codex:  ~/.codex/sessions/**/*.jsonl → CumulativeTokens → compute_delta() → UsageEntry
  └── Pi:     ~/.pi/agent/sessions/{project}/{session}.jsonl → PiUsage → UsageEntry (all 4 token fields)

UsageEntry.tokens: TokenCounts {
    input_tokens, output_tokens,
    cache_creation_tokens, cache_read_tokens
}

Aggregation → aggregation_types.rs (DailyUsage, SessionUsage, etc. all use TokenCounts)

Cost Calculation → CostCalculator::calculate_from_pricing() prices all 4 token fields

Display → output.rs shows "Cache Create" and "Cache Read" columns in detailed views
```

---

## Gaps and Issues

### 1. ⚠️ Codex: `cache_creation_tokens` is ALWAYS zero
- **Location**: `crates/ccstat-provider-codex/src/data_loader.rs` lines 251, 277
- **Issue**: `compute_delta()` always passes `0` for `cache_creation_tokens`. The `CumulativeTokens` struct has no `cache_creation` field at all.
- **Impact**: Codex logs may contain cached input tokens (mapped to `cache_read`), but any cache *creation* is silently lost.
- **Likely correct**: OpenAI's API doesn't distinguish cache creation vs cache read — it only reports `cached_input_tokens`. This mapping to `cache_read_tokens` is reasonable.

### 2. ⚠️ Codex: `cached_input_tokens` field name ambiguity
- **Location**: `crates/ccstat-provider-codex/src/data_loader.rs` lines 126-132
- **Issue**: The `CumulativeTokens` struct has both `cached_input_tokens` and `cache_read_tokens` (with alias `cache_read_input_tokens`). The code uses `if cached_input_tokens > 0 { cached_input_tokens } else { cache_read_tokens }` to pick the first non-zero value.
- **Risk**: If both fields are non-zero, `cached_input_tokens` wins. If these represent different things, this is wrong.

### 3. ⚠️ Zero-token skip may discard cache-only entries
- **Codex** (line 221): `if delta.input_tokens == 0 && delta.output_tokens == 0`
- **Pi** (line 184): `if usage.input == 0 && usage.output == 0`
- **Issue**: If a response has zero input/output tokens but has cache_read or cache_creation tokens, the entry is skipped entirely.
- **Real-world impact**: Likely minimal — cache tokens are usually accompanied by input tokens. But it's a correctness gap.

### 4. ✅ Pi Provider: Correctly handles all cache token fields
- `cache_read` → `cache_read_tokens` ✅
- `cache_write` → `cache_creation_tokens` ✅
- Pi log files use camelCase (`cacheRead`, `cacheWrite`) with `#[serde(rename_all = "camelCase")]` ✅

### 5. ✅ Pricing: Cache tokens are correctly priced
- `CostCalculator::calculate_from_pricing()` handles all 4 token types
- When `cache_*_cost` is `None`, those tokens contribute $0 to cost (correct fallback)
- OpenAI models in pricing.json have `cache_read_input_token_cost` but not `cache_creation_input_token_cost` (correct — OpenAI doesn't have that concept)

### 6. ⚠️ Pricing model matching for Codex
- Codex normalizes `gpt-5-codex` → `gpt-5` (line 287)
- Pricing has `openai/gpt-5` with `cache_read_input_token_cost: 1.25e-07`
- The fuzzy matching in `find_model_pricing()` should find this via partial match ("gpt-5" is contained in "openai/gpt-5")
- But `cache_creation_input_token_cost` is `None` for OpenAI models — cache creation tokens won't be priced (correct behavior for OpenAI)

---

## Start Here

**`crates/ccstat-provider-codex/src/data_loader.rs`** — This is the primary file that needs attention for cache token handling. The `compute_delta()` function (line 243) and `CumulativeTokens` struct (line 123) define how Codex tokens are parsed and mapped. Specifically:
- Line 251: `TokenCounts::new(..., 0, cache_read)` — cache_creation is hardcoded to 0
- Lines 246-250: The fallback logic between `cached_input_tokens` and `cache_read_tokens`

**`crates/ccstat-provider-pi/src/data_loader.rs`** — This is the reference for correct cache token mapping. The `PiUsage` struct (line 120) and the token mapping at lines 211-216 show the correct pattern.
