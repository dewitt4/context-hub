# Tavily JavaScript SDK Best Practices

## Search Optimization

### Query Guidelines

- Keep queries under **400 characters**. Think web search query, not LLM prompt.
- Break complex topics into sub-queries:

```javascript
const queries = [
  "Competitors of company ABC",
  "Financial performance of company ABC",
  "Recent developments of company ABC"
];

const responses = await Promise.all(queries.map(q => client.search(q)));
```

### Search Depth Tradeoffs

| Depth | Latency | Relevance | Content Type | Credits |
|-------|---------|-----------|-------------|---------|
| `basic` | Medium | High | NLP content summary | 1 |
| `advanced` | Higher | Highest | Reranked chunks | 2 |

- Use **`basic`** for general-purpose searches where a page summary suffices.
- Use **`advanced`** when you need highly targeted snippets aligned to the query.
- `chunksPerSource` controls how many chunks (max 500 chars each) are returned per result. Only works with `advanced`.

### autoParameters

When enabled, Tavily auto-configures search params based on query intent. Your explicit values override automatic ones.

```javascript
const response = await client.search("impact of AI in education policy", {
  autoParameters: true,
  searchDepth: "basic"  // Override to control cost
});
```

**Warning:** `autoParameters` may set `searchDepth` to `"advanced"` (2 credits). Set `searchDepth` explicitly to control cost.

### Exact Match

Use `exactMatch: true` with quoted phrases when you need verbatim matches:

```javascript
const response = await client.search('"John Smith" CEO Acme Corp', {
  exactMatch: true
});
```

Best for due diligence, data enrichment, and legal/compliance lookups. May return fewer results.

## Filtering Strategies

### By Date

```javascript
// Relative time range
const response = await client.search("latest ML trends", { timeRange: "month" });

// Specific date range
const response2 = await client.search("AI news", {
  startDate: "2025-01-01",
  endDate: "2025-02-01"
});
```

### By Topic

Set `topic: "news"` for news sources (includes `publishedDate` in results):

```javascript
const response = await client.search("What happened today in NY?", { topic: "news" });
```

### By Domain

```javascript
// Restrict to LinkedIn profiles
const response = await client.search("CEO background at Google", {
  includeDomains: ["linkedin.com/in"]
});

// Exclude irrelevant domains
const response2 = await client.search("US economy trends", {
  excludeDomains: ["espn.com", "vogue.com"]
});

// Boost results from a country
const response3 = await client.search("tech startup funding", { country: "united states" });
```

Keep domain lists short and relevant for best results.

### Post-Processing with Scores

```javascript
const filtered = response.results.filter(r => r.score > 0.7);
```

## Extract Best Practices

### When to Use Extract vs Search with rawContent

| Approach | When to Use |
|----------|-------------|
| `search({ includeRawContent: true })` | Quick prototyping, simple queries, single API call |
| `extract(urls, { query })` | You have specific URLs, need targeted extraction |

### Targeted Extraction with Query

Use `query` and `chunksPerSource` to get only relevant portions:

```javascript
const response = await client.extract(["https://example.com/long-article"], {
  query: "machine learning applications in healthcare",
  chunksPerSource: 3
});
```

This prevents context window explosion — returns max 500-char chunks instead of full pages.

### Extract Depth

| Depth | Use Case |
|-------|----------|
| `basic` (default) | Simple text, fast processing |
| `advanced` | JS-rendered pages, tables, structured data, embedded content |

`advanced` has higher success rate but increases latency and cost.

## Crawl Best Practices

### Start Conservative

Each level of `maxDepth` increases crawl time exponentially:

```javascript
// Start here
const response = await client.crawl("https://example.com", {
  maxDepth: 1, maxBreadth: 20, limit: 20
});

// Scale up only if needed
const response2 = await client.crawl("https://example.com", {
  maxDepth: 2, maxBreadth: 50, limit: 100
});
```

### Always Set a Limit

Never crawl without a `limit` — prevents runaway crawls and unexpected costs.

### Use Instructions for Focus

Without `instructions`, the crawl follows all links indiscriminately:

```javascript
const response = await client.crawl("https://docs.example.com", {
  instructions: "Find all API reference documentation",
  chunksPerSource: 3,
  maxDepth: 2
});
```

### Use Map Before Crawl

Discover site structure first, then crawl targeted paths:

```javascript
// 1. Map to discover structure
const mapResponse = await client.map("https://docs.example.com", { maxDepth: 2 });

// 2. Crawl with targeted paths
const response = await client.crawl("https://docs.example.com", {
  selectPaths: ["/api/.*", "/reference/.*"],
  maxDepth: 2
});
```

### Path Filtering

Use regex patterns to focus or exclude paths:

```javascript
const response = await client.crawl("https://example.com", {
  selectPaths: ["/blog/.*", "/docs/.*"],
  excludePaths: ["/private/.*", "/admin/.*", "/test/.*"]
});
```

## Common Patterns

### Concurrent Sub-Queries

```javascript
async function research(topic) {
  const subQueries = [
    `${topic} overview`,
    `${topic} recent developments`,
    `${topic} challenges and limitations`
  ];

  const responses = await Promise.allSettled(
    subQueries.map(q => client.search(q, { searchDepth: "advanced" }))
  );

  return responses
    .filter(r => r.status === "fulfilled")
    .flatMap(r => r.value.results);
}
```

### Score-Based Filtering Pipeline

```javascript
async function filteredExtraction(query) {
  // Search with many results
  const response = await client.search(query, {
    searchDepth: "advanced",
    maxResults: 20
  });

  // Filter by relevance score and deduplicate
  const urls = [...new Set(
    response.results
      .filter(r => r.score > 0.5)
      .map(r => r.url)
  )].slice(0, 20);

  // Extract from top URLs
  return client.extract(urls, {
    query,
    chunksPerSource: 3,
    extractDepth: "advanced"
  });
}
```

## Credit Usage

| Endpoint | Credits |
|----------|---------|
| Search (basic) | 1 per request |
| Search (advanced) | 2 per request |
| Extract (basic) | 1 per 5 successful URLs |
| Extract (advanced) | 2 per 5 successful URLs |
| Crawl | Varies by pages crawled |
| Map | 1 per request |
