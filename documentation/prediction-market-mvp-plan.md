# Prediction Market MVP Plan

## Goal

Build a small prediction-market matching service on top of Meilisearch without modifying core Meilisearch routes or ranking internals.

The MVP should accept:

- A plain-language question
- A prediction market URL
- A news article URL

The MVP should return:

- A finite list of the top 5-10 matching open prediction markets
- Each result should include at least the market title, platform, URL, and score

## High-level approach

Keep core Meilisearch unchanged and build a wrapper crate in this workspace.

Meilisearch already provides the retrieval primitives needed:

- Keyword search
- Semantic search
- Hybrid search
- Similar-document retrieval for already indexed documents

The main work for the MVP is:

- Defining a good market document schema
- Ingesting prediction market data into a single Meilisearch index
- Preprocessing URLs outside Meilisearch
- Calling the existing `/search` and `/similar` endpoints with the right payloads

## MVP scope

### In scope

- One index named `markets`
- One wrapper crate
- One initial market provider or one clean merged dataset
- Hybrid search for text input
- Exact lookup plus similar-market retrieval for known market URLs
- Basic news URL extraction followed by hybrid search
- A small offline evaluation harness

### Out of scope

- Server-side URL fetching inside Meilisearch
- Changes to Meilisearch ranking internals
- Multi-provider ingestion if one provider is enough for validation
- Full production-grade scraping
- Full entity extraction or LLM orchestration

## Repo changes

### Workspace change

Modify `Cargo.toml` to add a new workspace member:

- `crates/prediction-market-mvp`

### New crate

Create:

- `crates/prediction-market-mvp/Cargo.toml`
- `crates/prediction-market-mvp/src/main.rs`
- `crates/prediction-market-mvp/src/config.rs`
- `crates/prediction-market-mvp/src/models.rs`
- `crates/prediction-market-mvp/src/meili.rs`
- `crates/prediction-market-mvp/src/normalize.rs`
- `crates/prediction-market-mvp/src/extract.rs`
- `crates/prediction-market-mvp/src/match.rs`
- `crates/prediction-market-mvp/src/server.rs`
- `crates/prediction-market-mvp/src/ingest.rs`
- `crates/prediction-market-mvp/src/eval.rs`

### Data files

Create:

- `workloads/prediction_markets/markets.jsonl`
- `workloads/prediction_markets/eval.jsonl`

## Dependencies

Use these dependencies in `crates/prediction-market-mvp/Cargo.toml`:

- `tokio`
- `reqwest`
- `serde`
- `serde_json`
- `clap`
- `actix-web`
- `url`
- `regex`
- `anyhow`
- `tracing`
- `tracing-subscriber`

## CLI and runtime shape

The new crate should expose four commands:

- `init-index`
- `ingest`
- `serve`
- `eval`

### Command responsibilities

`init-index`

- Create the `markets` index if needed
- Apply index settings

`ingest`

- Read normalized or raw JSONL market data
- Normalize records into one common schema
- Upsert documents into Meilisearch

`serve`

- Start a small HTTP server
- Expose `POST /match`
- Expose `GET /health`

`eval`

- Load labeled eval cases
- Run the same matching pipeline used by `serve`
- Report `Hit@1`, `Recall@5`, and latency metrics

## Document schema

Use one document per market.

```json
{
  "id": "polymarket:12345",
  "platform": "polymarket",
  "market_url": "https://polymarket.com/event/...",
  "canonical_url": "https://polymarket.com/event/...",
  "question": "Will X happen before Y?",
  "description": "Market description",
  "resolution_criteria": "Resolves YES if ...",
  "status": "open",
  "close_time": 1770000000,
  "event_id": "event:x-y",
  "tags": ["politics"],
  "entity_names": ["X", "Y"]
}
```

### Rust type

Define this in `src/models.rs`:

```rust
pub struct MarketDocument {
    pub id: String,
    pub platform: String,
    pub market_url: String,
    pub canonical_url: String,
    pub question: String,
    pub description: Option<String>,
    pub resolution_criteria: Option<String>,
    pub status: String,
    pub close_time: Option<i64>,
    pub event_id: Option<String>,
    pub tags: Vec<String>,
    pub entity_names: Vec<String>,
}

pub struct MatchRequest {
    pub input: String,
    pub limit: Option<usize>,
}

pub struct MatchResponse {
    pub input_type: String,
    pub strategy: String,
    pub normalized_query: Option<String>,
    pub results: Vec<MatchedMarket>,
}

pub struct MatchedMarket {
    pub id: String,
    pub platform: String,
    pub market_url: String,
    pub question: String,
    pub status: String,
    pub score: Option<f64>,
}
```

## Index settings

Initialize the `markets` index with the following settings:

```json
{
  "displayedAttributes": [
    "id",
    "platform",
    "market_url",
    "canonical_url",
    "question",
    "description",
    "resolution_criteria",
    "status",
    "close_time",
    "event_id",
    "tags",
    "entity_names"
  ],
  "searchableAttributes": [
    "question",
    "description",
    "resolution_criteria",
    "tags",
    "entity_names"
  ],
  "filterableAttributes": [
    "status",
    "platform",
    "close_time",
    "tags",
    "event_id",
    "canonical_url",
    "market_url"
  ],
  "sortableAttributes": ["close_time"],
  "distinctAttribute": "event_id",
  "pagination": { "maxTotalHits": 100 },
  "embedders": {
    "default": {
      "source": "openAi",
      "model": "text-embedding-3-small",
      "documentTemplate": "{{doc.question}}\n{{doc.description}}\n{{doc.resolution_criteria}}\n{{doc.tags}}\n{{doc.entity_names}}"
    }
  }
}
```

### Why these settings

- `searchableAttributes` focus retrieval on semantic market content
- `filterableAttributes` support open-only filtering and exact URL lookup
- `distinctAttribute=event_id` reduces duplicate markets for the same event
- `pagination.maxTotalHits=100` keeps responses bounded
- The embedder `documentTemplate` gives semantic search enough context

## File-by-file implementation plan

### `src/main.rs`

Responsibilities:

- Parse CLI commands using `clap`
- Load config
- Dispatch to `init-index`, `ingest`, `serve`, or `eval`

### `src/config.rs`

Responsibilities:

- Define runtime config
- Read from environment variables

Recommended variables:

- `MEILI_URL`
- `MEILI_API_KEY`
- `MEILI_INDEX`
- `MVP_BIND_ADDR`
- `OPENAI_API_KEY`

### `src/models.rs`

Responsibilities:

- Define document schema
- Define request and response types
- Define eval row type

Suggested eval row:

```rust
pub struct EvalRow {
    pub input_type: String,
    pub input: String,
    pub expected_ids: Vec<String>,
}
```

### `src/meili.rs`

Responsibilities:

- Thin HTTP client over Meilisearch
- One method per operation used by the MVP

Required methods:

- `ensure_index`
- `update_settings`
- `add_documents`
- `search`
- `similar`
- `wait_for_task`

### `src/normalize.rs`

Responsibilities:

- Convert provider-specific market rows into `MarketDocument`
- Canonicalize URLs
- Enforce stable IDs

First-pass rule:

- Support one provider first, likely Polymarket
- If using a merged export, still centralize normalization here

### `src/extract.rs`

Responsibilities:

- Detect input type
- Normalize prediction market URLs
- Fetch and lightly extract article text from news URLs

Functions:

- `detect_input_type(input: &str) -> InputType`
- `normalize_market_url(url: &str) -> String`
- `extract_article_query(url: &str) -> Result<String>`

V1 extraction strategy:

- Page title
- `og:title`
- Meta description
- First one or two meaningful paragraphs
- Fallback to URL slug if extraction is weak

### `src/match.rs`

Responsibilities:

- Implement the main matching pipeline
- Route each input type to the right Meilisearch call

Core flow:

- Text input -> hybrid `/search`
- Market URL -> exact lookup -> `/similar`
- News URL -> external extraction -> hybrid `/search`

Also:

- Apply `status = open` filtering
- Deduplicate final results
- Return debug metadata such as `input_type`, `strategy`, and normalized query

### `src/server.rs`

Responsibilities:

- Start HTTP server
- Expose `POST /match`
- Expose `GET /health`

Suggested response:

```json
{
  "input_type": "news_url",
  "strategy": "article_extract_then_hybrid_search",
  "normalized_query": "Ukraine ceasefire talks resume after EU and US diplomatic push",
  "results": [
    {
      "id": "polymarket:123",
      "platform": "polymarket",
      "market_url": "https://...",
      "question": "Will a ceasefire be announced before ...?",
      "status": "open",
      "score": 0.82
    }
  ]
}
```

### `src/ingest.rs`

Responsibilities:

- Read `markets.jsonl`
- Normalize each input row
- Batch documents
- Upload to Meilisearch
- Wait for task completion

Batching recommendation:

- Use batches of 100-500 documents

### `src/eval.rs`

Responsibilities:

- Read `eval.jsonl`
- Run the same matching logic used in production
- Measure quality and latency

Metrics:

- `Hit@1`
- `Recall@5`
- Median latency

## Exact Meilisearch payloads

### 1. Create and configure the index

```json
PATCH /indexes/markets/settings
{
  "displayedAttributes": [
    "id",
    "platform",
    "market_url",
    "canonical_url",
    "question",
    "description",
    "resolution_criteria",
    "status",
    "close_time",
    "event_id",
    "tags",
    "entity_names"
  ],
  "searchableAttributes": [
    "question",
    "description",
    "resolution_criteria",
    "tags",
    "entity_names"
  ],
  "filterableAttributes": [
    "status",
    "platform",
    "close_time",
    "tags",
    "event_id",
    "canonical_url",
    "market_url"
  ],
  "sortableAttributes": ["close_time"],
  "distinctAttribute": "event_id",
  "pagination": { "maxTotalHits": 100 },
  "embedders": {
    "default": {
      "source": "openAi",
      "model": "text-embedding-3-small",
      "documentTemplate": "{{doc.question}}\n{{doc.description}}\n{{doc.resolution_criteria}}\n{{doc.tags}}\n{{doc.entity_names}}"
    }
  }
}
```

### 2. Ingest documents

```json
POST /indexes/markets/documents
[
  {
    "id": "polymarket:12345",
    "platform": "polymarket",
    "market_url": "https://polymarket.com/event/...",
    "canonical_url": "https://polymarket.com/event/...",
    "question": "Will X happen before Y?",
    "description": "Market description",
    "resolution_criteria": "Resolves YES if ...",
    "status": "open",
    "close_time": 1770000000,
    "event_id": "event:x-y",
    "tags": ["politics"],
    "entity_names": ["X", "Y"]
  }
]
```

### 3. Text question -> hybrid search

```json
POST /indexes/markets/search
{
  "q": "Will Trump visit China this year?",
  "filter": "status = open",
  "limit": 5,
  "attributesToRetrieve": ["id", "platform", "market_url", "question", "status"],
  "showRankingScore": true,
  "rankingScoreThreshold": 0.35,
  "hybrid": {
    "embedder": "default",
    "semanticRatio": 0.6
  }
}
```

### 4. Exact market URL lookup

Use filter-only lookup.

```json
POST /indexes/markets/search
{
  "filter": "canonical_url = \"https://polymarket.com/event/...\"",
  "limit": 1,
  "attributesToRetrieve": ["id", "platform", "market_url", "question", "status"]
}
```

### 5. Known market URL -> similar markets

```json
POST /indexes/markets/similar
{
  "id": "polymarket:12345",
  "embedder": "default",
  "limit": 5,
  "filter": "status = open",
  "showRankingScore": true,
  "rankingScoreThreshold": 0.45,
  "attributesToRetrieve": ["id", "platform", "market_url", "question", "status"]
}
```

### 6. News URL -> extracted summary -> hybrid search

Keep the extracted query short.

```json
POST /indexes/markets/search
{
  "q": "Ukraine ceasefire talks resume after EU and US diplomatic push",
  "filter": "status = open",
  "limit": 5,
  "attributesToRetrieve": ["id", "platform", "market_url", "question", "status"],
  "showRankingScore": true,
  "rankingScoreThreshold": 0.30,
  "hybrid": {
    "embedder": "default",
    "semanticRatio": 0.7
  }
}
```

## Matching logic by input type

### Plain text

1. Detect non-URL input
2. Send hybrid `/search`
3. Filter to open markets
4. Return top 5-10 hits

### Prediction market URL

1. Normalize URL
2. Exact-match `canonical_url`
3. If found, call `/similar`
4. If not found, derive text from URL slug or title and fall back to hybrid `/search`

### News URL

1. Fetch page outside Meilisearch
2. Extract short title and summary
3. Call hybrid `/search`
4. Return top 5-10 open markets

## Initial tuning values

Use these as the first pass:

- Text input: `semanticRatio = 0.6`, `rankingScoreThreshold = 0.35`
- Similar-doc lookup: `rankingScoreThreshold = 0.45`
- News URL summary: `semanticRatio = 0.7`, `rankingScoreThreshold = 0.30`

These should be tuned after the first evaluation pass.

## Evaluation dataset

Create `workloads/prediction_markets/eval.jsonl` with rows like:

```json
{"input_type":"text","input":"Will the Fed cut rates by June 2026?","expected_ids":["polymarket:1"]}
{"input_type":"market_url","input":"https://polymarket.com/event/xyz","expected_ids":["polymarket:2","manifold:8"]}
{"input_type":"news_url","input":"https://example.com/article","expected_ids":["polymarket:9"]}
```

### Recommended eval mix

- 40 plain text questions
- 30 prediction market URLs
- 30 news URLs

### Metrics

- `Hit@1`
- `Recall@5`
- Median latency

## Implementation order

1. Add the new crate to the workspace
2. Implement config loading and the Meilisearch client
3. Implement `init-index`
4. Define `MarketDocument`
5. Implement ingestion for one provider or one clean merged dataset
6. Implement plain-text matching using hybrid `/search`
7. Implement known market URL lookup plus `/similar`
8. Implement basic news URL extraction plus hybrid `/search`
9. Implement `eval`
10. Tune thresholds and document template using eval results

## Definition of done

The MVP is complete when:

- `init-index` creates and configures the `markets` index
- `ingest` loads a real market dataset into Meilisearch
- `serve` returns top 5 matches for text questions
- `serve` returns top 5 similar markets for known market URLs already in the index
- `serve` returns top 5 candidate markets for news URLs using lightweight article extraction
- `eval` runs over at least 100 labeled inputs and reports quality metrics

## Success criteria

Use these first-pass targets:

- `Recall@5 >= 0.70` on plain text questions
- `Hit@1 >= 0.80` on known market URLs already present in the index
- Median latency under 1 second for plain text and known market URL inputs

## What not to do yet

- Do not add URL fetching to core Meilisearch routes
- Do not modify hybrid ranking internals
- Do not optimize for many providers before one provider works well
- Do not add a large LLM pipeline before baseline retrieval is measured

## Recommended next step after writing this plan

Implement the crate skeleton first:

- `init-index`
- `ingest`
- `serve` with plain-text matching only

Once that path works end-to-end, add:

- Market URL lookup plus `/similar`
- News URL extraction
- Offline evaluation
