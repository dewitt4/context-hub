---
name: sdk
description: "Web search API for AI agents with search, extract, crawl, map, and research endpoints"
metadata:
  languages: "python"
  versions: "0.7.23"
  revision: 1
  updated-on: "2026-03-11"
  source: official
  tags: "tavily,search,extract,crawl,research,ai,agents,rag"
---
# Tavily Python SDK

Web search API built for AI agents and RAG pipelines. Provides search, extract, crawl, map, and research through a single SDK.

**Package:** `tavily-python` on PyPI
**Repo:** github.com/tavily-ai/tavily-python

## Installation

```bash
pip install tavily-python
```

If using `uv`, you can add the package to your project with:
```bash
uv add tavily-python
```

## Initialization

Set `TAVILY_API_KEY` in your environment, or pass it directly.

### Synchronous Client

```python
from tavily import TavilyClient

client = TavilyClient(api_key="tvly-YOUR_API_KEY")
```

### Asynchronous Client

```python
from tavily import AsyncTavilyClient

client = AsyncTavilyClient(api_key="tvly-YOUR_API_KEY")
```

### Project Tracking

Attach a project ID to organize usage across projects:

```python
client = TavilyClient(api_key="tvly-YOUR_API_KEY", project_id="my-project")
```

Or set `TAVILY_PROJECT` environment variable.

## Search

The core endpoint. Returns ranked web results with content snippets.

```python
response = client.search("latest developments in quantum computing")
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `query` **(required)** | `str` | — | Search query (keep under 400 chars) |
| `search_depth` | `str` | `"basic"` | `"basic"` or `"advanced"`. Advanced returns reranked chunks (2 credits) |
| `topic` | `str` | `"general"` | `"general"`, `"news"`, or `"finance"` |
| `max_results` | `int` | `5` | Number of results (0–20) |
| `include_answer` | `bool`/`str` | `False` | `True`/`"basic"` for quick answer, `"advanced"` for detailed |
| `include_raw_content` | `bool`/`str` | `False` | `True`/`"markdown"` for markdown, `"text"` for plain text |
| `include_images` | `bool` | `False` | Include query-related images |
| `include_domains` | `list[str]` | `[]` | Restrict to specific domains (max 300) |
| `exclude_domains` | `list[str]` | `[]` | Exclude specific domains (max 150) |
| `time_range` | `str` | — | `"day"`, `"week"`, `"month"`, `"year"` |
| `start_date` | `str` | — | Filter from date (`YYYY-MM-DD`) |
| `end_date` | `str` | — | Filter until date (`YYYY-MM-DD`) |
| `chunks_per_source` | `int` | `3` | Chunks per result (only with `search_depth="advanced"`) |
| `country` | `str` | — | Boost results from a country (general topic only) |
| `auto_parameters` | `bool` | `False` | Auto-configure params based on query intent |
| `exact_match` | `bool` | `False` | Require exact quoted phrases in results |

### Response

```python
{
    "query": "latest developments in quantum computing",
    "results": [
        {
            "title": "...",
            "url": "https://...",
            "content": "Most relevant snippet...",
            "score": 0.99,
            "raw_content": "...",         # if include_raw_content
            "published_date": "...",       # if topic="news"
        }
    ],
    "answer": "...",          # if include_answer
    "images": [...],          # if include_images
    "response_time": 1.09,
    "request_id": "..."
}
```

### Advanced Search Example

```python
response = client.search(
    query="How many countries use Daylight Saving Time?",
    search_depth="advanced",
    max_results=10,
    include_answer="advanced",
    include_raw_content=True,
    chunks_per_source=3
)

print(response["answer"])
for result in response["results"]:
    print(f"{result['title']} ({result['score']:.2f})")
    print(f"  {result['url']}")
```

### Domain Filtering

```python
# Restrict to specific sources
response = client.search(
    query="CEO background at Google",
    include_domains=["linkedin.com/in"]
)

# Exclude irrelevant domains
response = client.search(
    query="US economy trends",
    exclude_domains=["espn.com", "vogue.com"]
)
```

### News Search

```python
response = client.search(
    query="AI regulation updates",
    topic="news",
    time_range="week",
    max_results=10
)

for result in response["results"]:
    print(f"{result['title']} - {result.get('published_date', 'N/A')}")
```

## Extract

Extract content from specific URLs. Returns cleaned markdown or plain text.

```python
response = client.extract(urls=["https://en.wikipedia.org/wiki/Quantum_computing"])
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `urls` **(required)** | `list[str]` | — | URLs to extract (max 20) |
| `extract_depth` | `str` | `"basic"` | `"basic"` or `"advanced"` (JS-heavy pages, tables) |
| `format` | `str` | `"markdown"` | `"markdown"` or `"text"` |
| `query` | `str` | — | Rerank chunks by relevance to this query |
| `chunks_per_source` | `int` | `3` | Chunks per URL (1–5, requires `query`) |
| `timeout` | `float` | — | Timeout in seconds (1.0–60.0) |

### Example with Query Filtering

```python
response = client.extract(
    urls=[
        "https://example.com/ml-healthcare",
        "https://example.com/ai-diagnostics"
    ],
    query="AI diagnostic tools accuracy",
    chunks_per_source=2,
    extract_depth="advanced"
)

for result in response["results"]:
    print(f"URL: {result['url']}")
    print(f"Content: {result['raw_content'][:200]}...")

for failed in response["failed_results"]:
    print(f"Failed: {failed['url']} - {failed['error']}")
```

## Crawl

Intelligently traverse a website and extract content from discovered pages.

```python
response = client.crawl(
    url="https://docs.tavily.com",
    instructions="Find all pages about the Python SDK"
)
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` **(required)** | `str` | — | Starting URL |
| `max_depth` | `int` | `1` | Levels deep to crawl (each level increases time exponentially) |
| `max_breadth` | `int` | `20` | Max links to follow per page |
| `limit` | `int` | `50` | Total max pages to crawl |
| `instructions` | `str` | — | Natural language guidance to focus the crawl |
| `chunks_per_source` | `int` | — | Content chunks per page (requires `instructions`) |
| `select_paths` | `list[str]` | — | Regex patterns for paths to include |
| `exclude_paths` | `list[str]` | — | Regex patterns for paths to exclude |
| `select_domains` | `list[str]` | — | Regex patterns for domains to include |
| `exclude_domains` | `list[str]` | — | Regex patterns for domains to exclude |
| `extract_depth` | `str` | `"basic"` | `"basic"` or `"advanced"` |

### Focused Crawl Example

```python
response = client.crawl(
    url="https://docs.example.com",
    max_depth=2,
    limit=100,
    instructions="Find all API reference documentation",
    select_paths=["/docs/.*", "/api/.*"],
    exclude_paths=["/private/.*", "/admin/.*"],
    extract_depth="advanced"
)

for page in response["results"]:
    print(f"{page['url']}: {len(page['raw_content'])} chars")
```

## Map

Discover site structure without extracting content. Returns a list of URLs.

```python
response = client.map(
    url="https://docs.tavily.com",
    instructions="Find all pages on the Python SDK"
)
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` **(required)** | `str` | — | Starting URL |
| `max_depth` | `int` | `1` | Levels deep to map |
| `max_breadth` | `int` | `20` | Max links per page |
| `limit` | `int` | `50` | Total max URLs |
| `instructions` | `str` | — | Focus the mapping with natural language |
| `select_paths` | `list[str]` | — | Regex path patterns to include |
| `exclude_paths` | `list[str]` | — | Regex path patterns to exclude |

### Response

```python
{
    "base_url": "https://docs.tavily.com",
    "results": [
        "https://docs.tavily.com/sdk/python/reference",
        "https://docs.tavily.com/sdk/python/quick-start"
    ],
    "response_time": 8.43,
    "request_id": "..."
}
```

**Tip:** Use Map first to discover structure, then Crawl with discovered paths for focused extraction.

## Async Usage

Use `AsyncTavilyClient` for concurrent requests:

```python
import asyncio
from tavily import AsyncTavilyClient

client = AsyncTavilyClient(api_key="tvly-YOUR_API_KEY")

async def parallel_search():
    queries = ["latest AI trends", "quantum computing breakthroughs"]
    responses = await asyncio.gather(
        *(client.search(q) for q in queries),
        return_exceptions=True
    )
    for response in responses:
        if isinstance(response, Exception):
            print(f"Failed: {response}")
        else:
            print(f"Results: {len(response['results'])}")

asyncio.run(parallel_search())
```

## Search Then Extract Pattern

A common RAG pattern: search to find URLs, then extract full content from the best ones.

```python
import asyncio
from tavily import AsyncTavilyClient

client = AsyncTavilyClient(api_key="tvly-YOUR_API_KEY")

async def search_then_extract(topic):
    # 1. Search for relevant URLs
    search_response = await client.search(
        query=topic,
        search_depth="advanced",
        max_results=10
    )

    # 2. Filter by relevance score
    urls = [
        r["url"] for r in search_response["results"]
        if r["score"] > 0.5
    ][:20]

    # 3. Extract full content
    extract_response = await client.extract(
        urls=urls,
        query=topic,
        chunks_per_source=3,
        extract_depth="advanced"
    )

    return extract_response["results"]

results = asyncio.run(search_then_extract("AI in healthcare diagnostics"))
```

## Error Handling

```python
from tavily import TavilyClient, MissingAPIKeyError, InvalidAPIKeyError, UsageLimitExceededError

try:
    client = TavilyClient(api_key="tvly-YOUR_API_KEY")
    response = client.search("test query")
except MissingAPIKeyError:
    print("API key not provided")
except InvalidAPIKeyError:
    print("Invalid API key")
except UsageLimitExceededError:
    print("API credit limit exceeded")
```
