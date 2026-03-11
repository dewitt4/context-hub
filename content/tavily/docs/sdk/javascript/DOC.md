---
name: sdk
description: "Web search API for AI agents with search, extract, crawl, map, and research endpoints"
metadata:
  languages: "javascript"
  versions: "0.7.2"
  revision: 1
  updated-on: "2026-03-11"
  source: maintainer
  tags: "tavily,search,extract,crawl,research,ai,agents,rag"
---
# Tavily JavaScript SDK

Web search API built for AI agents and RAG pipelines. Provides search, extract, crawl, map, and research through a single SDK. The client is async by default.

**Package:** `@tavily/core` on npm
**Repo:** github.com/tavily-ai/tavily-js

## Installation

```bash
npm install @tavily/core
```

## Initialization

```javascript
const { tavily } = require("@tavily/core");

const client = tavily({ apiKey: "tvly-YOUR_API_KEY" });
```

Or with ES modules:

```javascript
import { tavily } from "@tavily/core";

const client = tavily({ apiKey: "tvly-YOUR_API_KEY" });
```

### Project Tracking

Attach a project ID to organize usage:

```javascript
const client = tavily({
  apiKey: "tvly-YOUR_API_KEY",
  projectId: "my-project"
});
```

Or set `TAVILY_PROJECT` environment variable.

## Search

The core endpoint. Returns ranked web results with content snippets.

```javascript
const response = await client.search("latest developments in quantum computing");
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `query` **(required)** | `string` | — | Search query (keep under 400 chars) |
| `searchDepth` | `string` | `"basic"` | `"basic"` or `"advanced"`. Advanced returns reranked chunks (2 credits) |
| `topic` | `string` | `"general"` | `"general"`, `"news"`, or `"finance"` |
| `maxResults` | `number` | `5` | Number of results (0–20) |
| `includeAnswer` | `boolean`/`string` | `false` | `true`/`"basic"` for quick answer, `"advanced"` for detailed |
| `includeRawContent` | `boolean`/`string` | `false` | `true`/`"markdown"` for markdown, `"text"` for plain text |
| `includeImages` | `boolean` | `false` | Include query-related images |
| `includeDomains` | `string[]` | `[]` | Restrict to specific domains (max 300) |
| `excludeDomains` | `string[]` | `[]` | Exclude specific domains (max 150) |
| `timeRange` | `string` | — | `"day"`, `"week"`, `"month"`, `"year"` |
| `startDate` | `string` | — | Filter from date (`YYYY-MM-DD`) |
| `endDate` | `string` | — | Filter until date (`YYYY-MM-DD`) |
| `chunksPerSource` | `number` | `3` | Chunks per result (only with `searchDepth: "advanced"`) |
| `country` | `string` | — | Boost results from a country (general topic only) |
| `autoParameters` | `boolean` | `false` | Auto-configure params based on query intent |
| `exactMatch` | `boolean` | `false` | Require exact quoted phrases in results |

### Response

```javascript
{
  query: "latest developments in quantum computing",
  results: [
    {
      title: "...",
      url: "https://...",
      content: "Most relevant snippet...",
      score: 0.99,
      rawContent: "...",          // if includeRawContent
      publishedDate: "...",       // if topic="news"
    }
  ],
  answer: "...",          // if includeAnswer
  images: [...],          // if includeImages
  responseTime: 1.09,
  requestId: "..."
}
```

### Advanced Search Example

```javascript
const response = await client.search("How many countries use Monday.com?", {
  searchDepth: "advanced",
  maxResults: 10,
  includeAnswer: "advanced",
  includeRawContent: true,
  chunksPerSource: 3
});

console.log(response.answer);
for (const result of response.results) {
  console.log(`${result.title} (${result.score.toFixed(2)})`);
  console.log(`  ${result.url}`);
}
```

### Domain Filtering

```javascript
// Restrict to specific sources
const response = await client.search("CEO background at Google", {
  includeDomains: ["linkedin.com/in"]
});

// Exclude irrelevant domains
const response2 = await client.search("US economy trends", {
  excludeDomains: ["espn.com", "vogue.com"]
});
```

### News Search

```javascript
const response = await client.search("AI regulation updates", {
  topic: "news",
  timeRange: "week",
  maxResults: 10
});

for (const result of response.results) {
  console.log(`${result.title} - ${result.publishedDate || "N/A"}`);
}
```

## Extract

Extract content from specific URLs. Returns cleaned markdown or plain text.

```javascript
const response = await client.extract(["https://en.wikipedia.org/wiki/Quantum_computing"]);
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `urls` **(required)** | `string[]` | — | URLs to extract (max 20) |
| `extractDepth` | `string` | `"basic"` | `"basic"` or `"advanced"` (JS-heavy pages, tables) |
| `format` | `string` | `"markdown"` | `"markdown"` or `"text"` |
| `query` | `string` | — | Rerank chunks by relevance to this query |
| `chunksPerSource` | `number` | `3` | Chunks per URL (1–5, requires `query`) |
| `timeout` | `number` | — | Timeout in seconds (1.0–60.0) |

### Example with Query Filtering

```javascript
const response = await client.extract(
  ["https://example.com/ml-healthcare", "https://example.com/ai-diagnostics"],
  {
    query: "AI diagnostic tools accuracy",
    chunksPerSource: 2,
    extractDepth: "advanced"
  }
);

for (const result of response.results) {
  console.log(`URL: ${result.url}`);
  console.log(`Content: ${result.rawContent.slice(0, 200)}...`);
}

for (const failed of response.failedResults) {
  console.log(`Failed: ${failed.url} - ${failed.error}`);
}
```

## Crawl

Intelligently traverse a website and extract content from discovered pages.

```javascript
const response = await client.crawl("https://docs.tavily.com", {
  instructions: "Find all pages about the Python SDK"
});
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` **(required)** | `string` | — | Starting URL |
| `maxDepth` | `number` | `1` | Levels deep to crawl (each level increases time exponentially) |
| `maxBreadth` | `number` | `20` | Max links to follow per page |
| `limit` | `number` | `50` | Total max pages to crawl |
| `instructions` | `string` | — | Natural language guidance to focus the crawl |
| `chunksPerSource` | `number` | — | Content chunks per page (requires `instructions`) |
| `selectPaths` | `string[]` | — | Regex patterns for paths to include |
| `excludePaths` | `string[]` | — | Regex patterns for paths to exclude |
| `selectDomains` | `string[]` | — | Regex patterns for domains to include |
| `excludeDomains` | `string[]` | — | Regex patterns for domains to exclude |
| `extractDepth` | `string` | `"basic"` | `"basic"` or `"advanced"` |

### Focused Crawl Example

```javascript
const response = await client.crawl("https://docs.example.com", {
  maxDepth: 2,
  limit: 100,
  instructions: "Find all API reference documentation",
  selectPaths: ["/docs/.*", "/api/.*"],
  excludePaths: ["/private/.*", "/admin/.*"],
  extractDepth: "advanced"
});

for (const page of response.results) {
  console.log(`${page.url}: ${page.rawContent.length} chars`);
}
```

## Map

Discover site structure without extracting content. Returns a list of URLs.

```javascript
const response = await client.map("https://docs.tavily.com", {
  instructions: "Find all pages on the Python SDK"
});
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` **(required)** | `string` | — | Starting URL |
| `maxDepth` | `number` | `1` | Levels deep to map |
| `maxBreadth` | `number` | `20` | Max links per page |
| `limit` | `number` | `50` | Total max URLs |
| `instructions` | `string` | — | Focus the mapping with natural language |
| `selectPaths` | `string[]` | — | Regex path patterns to include |
| `excludePaths` | `string[]` | — | Regex path patterns to exclude |

### Response

```javascript
{
  baseUrl: "https://docs.tavily.com",
  results: [
    "https://docs.tavily.com/sdk/python/reference",
    "https://docs.tavily.com/sdk/python/quick-start"
  ],
  responseTime: 8.43,
  requestId: "..."
}
```

**Tip:** Use Map first to discover structure, then Crawl with discovered paths for focused extraction.

## Search Then Extract Pattern

A common RAG pattern: search to find URLs, then extract full content from the best ones.

```javascript
async function searchThenExtract(topic) {
  // 1. Search for relevant URLs
  const searchResponse = await client.search(topic, {
    searchDepth: "advanced",
    maxResults: 10
  });

  // 2. Filter by relevance score
  const urls = searchResponse.results
    .filter(r => r.score > 0.5)
    .map(r => r.url)
    .slice(0, 20);

  // 3. Extract full content
  const extractResponse = await client.extract(urls, {
    query: topic,
    chunksPerSource: 3,
    extractDepth: "advanced"
  });

  return extractResponse.results;
}

const results = await searchThenExtract("AI in healthcare diagnostics");
```

## Parallel Searches

Run multiple independent searches concurrently:

```javascript
const queries = ["latest AI trends", "quantum computing breakthroughs"];

const responses = await Promise.allSettled(
  queries.map(q => client.search(q))
);

for (const response of responses) {
  if (response.status === "fulfilled") {
    console.log(`Results: ${response.value.results.length}`);
  } else {
    console.log(`Failed: ${response.reason}`);
  }
}
```

## Error Handling

```javascript
try {
  const response = await client.search("test query");
} catch (error) {
  if (error.message.includes("API key")) {
    console.log("Invalid or missing API key");
  } else if (error.message.includes("rate limit")) {
    console.log("Rate limit exceeded, retry later");
  } else {
    console.log(`Error: ${error.message}`);
  }
}
```
