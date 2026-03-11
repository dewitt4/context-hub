# Tavily Python SDK Best Practices

## Search Optimization

### Query Guidelines

- Keep queries under **400 characters**. Think web search query, not LLM prompt.
- Break complex topics into sub-queries:

```python
# Instead of one massive query:
queries = [
    "Competitors of company ABC",
    "Financial performance of company ABC",
    "Recent developments of company ABC"
]

responses = await asyncio.gather(*(client.search(q) for q in queries))
```

### Search Depth Tradeoffs

| Depth | Latency | Relevance | Content Type | Credits |
|-------|---------|-----------|-------------|---------|
| `basic` | Medium | High | NLP content summary | 1 |
| `advanced` | Higher | Highest | Reranked chunks | 2 |

- Use **`basic`** for general-purpose searches where a page summary suffices.
- Use **`advanced`** when you need highly targeted snippets aligned to the query.
- `chunks_per_source` controls how many chunks (max 500 chars each) are returned per result. Only works with `advanced`.

### auto_parameters

When enabled, Tavily auto-configures search params based on query intent. Your explicit values override automatic ones.

```python
response = client.search(
    query="impact of AI in education policy",
    auto_parameters=True,
    search_depth="basic"  # Override to control cost
)
```

**Warning:** `auto_parameters` may set `search_depth` to `"advanced"` (2 credits). Set `search_depth` explicitly to control cost.

### Exact Match

Use `exact_match=True` with quoted phrases when you need verbatim matches:

```python
response = client.search(
    query='"John Smith" CEO Acme Corp',
    exact_match=True
)
```

Best for due diligence, data enrichment, and legal/compliance lookups. May return fewer results.

## Filtering Strategies

### By Date

```python
# Relative time range
response = client.search("latest ML trends", time_range="month")

# Specific date range
response = client.search("AI news", start_date="2025-01-01", end_date="2025-02-01")
```

### By Topic

Set `topic="news"` for news sources (includes `published_date` in results):

```python
response = client.search("What happened today in NY?", topic="news")
```

### By Domain

```python
# Restrict to LinkedIn profiles
response = client.search("CEO background at Google", include_domains=["linkedin.com/in"])

# Exclude irrelevant domains
response = client.search("US economy trends", exclude_domains=["espn.com", "vogue.com"])

# Boost results from a country
response = client.search("tech startup funding", country="united states")

# Wildcard domain patterns
response = client.search("AI news", include_domains=["*.com"], exclude_domains=["example.com"])
```

Keep domain lists short and relevant for best results.

### Post-Processing with Scores

```python
# Filter results by relevance score
filtered = [r for r in response["results"] if r["score"] > 0.7]
```

## Extract Best Practices

### When to Use Extract vs Search with raw_content

| Approach | When to Use |
|----------|-------------|
| `search(include_raw_content=True)` | Quick prototyping, simple queries, single API call |
| `extract(urls=[...])` | You have specific URLs, need targeted extraction with `query` |

### Targeted Extraction with Query

Use `query` and `chunks_per_source` to get only relevant portions:

```python
response = client.extract(
    urls=["https://example.com/long-article"],
    query="machine learning applications in healthcare",
    chunks_per_source=3
)
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

Each level of `max_depth` increases crawl time exponentially:

```python
# Start here
response = client.crawl("https://example.com", max_depth=1, max_breadth=20, limit=20)

# Scale up only if needed
response = client.crawl("https://example.com", max_depth=2, max_breadth=50, limit=100)
```

### Always Set a Limit

Never crawl without a `limit` — prevents runaway crawls and unexpected costs.

### Use Instructions for Focus

Without `instructions`, the crawl follows all links indiscriminately:

```python
# Focused crawl — much more efficient
response = client.crawl(
    url="https://docs.example.com",
    instructions="Find all API reference documentation",
    chunks_per_source=3,
    max_depth=2
)
```

### Use Map Before Crawl

Discover site structure first, then crawl targeted paths:

```python
# 1. Map to discover structure
map_response = client.map("https://docs.example.com", max_depth=2)

# 2. Crawl with targeted paths
response = client.crawl(
    url="https://docs.example.com",
    select_paths=["/api/.*", "/reference/.*"],
    max_depth=2
)
```

### Path Filtering

Use regex patterns to focus or exclude paths:

```python
response = client.crawl(
    url="https://example.com",
    select_paths=["/blog/.*", "/docs/.*"],
    exclude_paths=["/private/.*", "/admin/.*", "/test/.*"]
)
```

## Common Patterns

### Concurrent Sub-Queries

```python
import asyncio
from tavily import AsyncTavilyClient

client = AsyncTavilyClient(api_key="tvly-YOUR_API_KEY")

async def research(topic):
    sub_queries = [
        f"{topic} overview",
        f"{topic} recent developments",
        f"{topic} challenges and limitations"
    ]
    responses = await asyncio.gather(
        *(client.search(q, search_depth="advanced") for q in sub_queries),
        return_exceptions=True
    )
    results = []
    for resp in responses:
        if not isinstance(resp, Exception):
            results.extend(resp["results"])
    return results
```

### Score-Based Filtering Pipeline

```python
async def filtered_extraction(query):
    # Search with many results
    response = await client.search(query, search_depth="advanced", max_results=20)

    # Filter by relevance score
    urls = [r["url"] for r in response["results"] if r["score"] > 0.5]

    # Deduplicate
    urls = list(dict.fromkeys(urls))[:20]

    # Extract from top URLs
    return await client.extract(
        urls=urls,
        query=query,
        chunks_per_source=3,
        extract_depth="advanced"
    )
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
