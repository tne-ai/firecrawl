# Search Enhancement Implementation Plan

## Executive Summary
Enhance the search functionality by implementing multiple search engine support, intelligent result aggregation, fallback mechanisms, and avoiding basic Google scraping. This plan addresses the current limitation of falling back to basic Google scraping when SEARXNG_ENDPOINT isn't configured.

## Current State Analysis
- **Current Implementation**: Falls back to basic Google scraping when SEARXNG_ENDPOINT isn't configured
- **Limitations**:
  - Single search engine dependency
  - No result deduplication
  - No intelligent ranking
  - High risk of blocking from Google
  - No alternative search APIs

## Recommended Search Stack

### Primary Solutions

#### 1. SearXNG (Self-Hosted Meta Search)
- **Advantages**: Privacy-focused, aggregates 70+ search engines
- **Setup**: Docker-based deployment
- **Cost**: Free (self-hosted)

#### 2. Search APIs
- **Serper API**: $50/month for 2,500 searches
- **SerpAPI**: $75/month for 5,000 searches
- **ScraperAPI Search**: $49/month with proxy rotation
- **Brave Search API**: 2,000 free searches/month

#### 3. Alternative Search Engines
- **DuckDuckGo**: Privacy-focused, scrapable
- **Bing**: Microsoft's search with API access
- **Yandex**: Russian search engine, less restrictive
- **Qwant**: European privacy-focused search

## Implementation Architecture

### Phase 1: Multi-Engine Search Service (Week 1)

#### 1.1 Core Search Manager

**File**: `apps/api/src/services/search/SearchManager.ts`

```typescript
export interface SearchQuery {
  query: string;
  limit?: number;
  offset?: number;
  language?: string;
  country?: string;
  timeRange?: 'day' | 'week' | 'month' | 'year' | 'all';
  safeSearch?: boolean;
  type?: 'web' | 'image' | 'video' | 'news';
}

export interface SearchResult {
  title: string;
  url: string;
  snippet: string;
  position: number;
  source: string;
  timestamp?: Date;
  metadata?: Record<string, any>;
}

export interface SearchResponse {
  results: SearchResult[];
  totalResults?: number;
  searchTime: number;
  sources: string[];
  query: SearchQuery;
}

export class SearchManager {
  private engines: Map<string, SearchEngine> = new Map();
  private primaryEngine: string;
  private resultAggregator: ResultAggregator;
  private cache: SearchCache;

  constructor() {
    this.initializeEngines();
    this.resultAggregator = new ResultAggregator();
    this.cache = new SearchCache();
    this.primaryEngine = process.env.PRIMARY_SEARCH_ENGINE || 'searxng';
  }

  private initializeEngines(): void {
    // Initialize SearXNG
    if (process.env.SEARXNG_ENDPOINT) {
      this.engines.set('searxng', new SearXNGEngine({
        endpoint: process.env.SEARXNG_ENDPOINT,
        apiKey: process.env.SEARXNG_API_KEY
      }));
    }

    // Initialize Serper API
    if (process.env.SERPER_API_KEY) {
      this.engines.set('serper', new SerperEngine({
        apiKey: process.env.SERPER_API_KEY
      }));
    }

    // Initialize Brave Search
    if (process.env.BRAVE_SEARCH_API_KEY) {
      this.engines.set('brave', new BraveSearchEngine({
        apiKey: process.env.BRAVE_SEARCH_API_KEY
      }));
    }

    // Initialize SerpAPI
    if (process.env.SERPAPI_KEY) {
      this.engines.set('serpapi', new SerpAPIEngine({
        apiKey: process.env.SERPAPI_KEY
      }));
    }

    // Initialize DuckDuckGo scraper (no API key needed)
    this.engines.set('duckduckgo', new DuckDuckGoScraper());

    // Initialize Bing
    if (process.env.BING_API_KEY) {
      this.engines.set('bing', new BingSearchEngine({
        apiKey: process.env.BING_API_KEY
      }));
    }
  }

  async search(query: SearchQuery): Promise<SearchResponse> {
    // Check cache first
    const cached = await this.cache.get(query);
    if (cached) {
      return cached;
    }

    const startTime = Date.now();
    let results: SearchResult[] = [];
    let sources: string[] = [];

    try {
      // Try primary engine first
      const primaryResults = await this.searchWithEngine(this.primaryEngine, query);
      results = primaryResults.results;
      sources.push(this.primaryEngine);
    } catch (primaryError) {
      logger.warn('Primary search engine failed', {
        engine: this.primaryEngine,
        error: primaryError
      });

      // Fallback to alternative engines
      results = await this.searchWithFallback(query);
      sources = this.getUsedEngines(results);
    }

    // Aggregate and rank results if multiple sources
    if (sources.length > 1) {
      results = await this.resultAggregator.aggregate(results);
    }

    const response: SearchResponse = {
      results: results.slice(0, query.limit || 10),
      totalResults: results.length,
      searchTime: Date.now() - startTime,
      sources,
      query
    };

    // Cache successful results
    await this.cache.set(query, response);

    return response;
  }

  private async searchWithEngine(
    engineName: string,
    query: SearchQuery
  ): Promise<SearchResponse> {
    const engine = this.engines.get(engineName);
    if (!engine) {
      throw new Error(`Search engine ${engineName} not configured`);
    }

    return engine.search(query);
  }

  private async searchWithFallback(query: SearchQuery): Promise<SearchResult[]> {
    const fallbackOrder = this.determineFallbackOrder();
    const results: SearchResult[] = [];

    for (const engineName of fallbackOrder) {
      try {
        const engineResults = await this.searchWithEngine(engineName, query);
        results.push(...engineResults.results);

        if (results.length >= (query.limit || 10)) {
          break;
        }
      } catch (error) {
        logger.warn('Fallback engine failed', {
          engine: engineName,
          error
        });
        continue;
      }
    }

    if (results.length === 0) {
      throw new Error('All search engines failed');
    }

    return results;
  }

  private determineFallbackOrder(): string[] {
    // Order by reliability and cost
    const order = [
      'searxng',      // Free, self-hosted
      'brave',        // Free tier available
      'duckduckgo',   // Free scraping
      'serper',       // Paid API
      'serpapi',      // Paid API
      'bing'          // Microsoft API
    ];

    return order.filter(engine => this.engines.has(engine));
  }

  private getUsedEngines(results: SearchResult[]): string[] {
    const sources = new Set<string>();
    results.forEach(result => sources.add(result.source));
    return Array.from(sources);
  }
}
```

#### 1.2 Search Engine Implementations

**File**: `apps/api/src/services/search/engines/SearXNGEngine.ts`

```typescript
export class SearXNGEngine implements SearchEngine {
  constructor(private config: { endpoint: string; apiKey?: string }) {}

  async search(query: SearchQuery): Promise<SearchResponse> {
    const params = new URLSearchParams({
      q: query.query,
      format: 'json',
      categories: this.mapType(query.type),
      language: query.language || 'en',
      time_range: query.timeRange || '',
      safesearch: query.safeSearch ? '2' : '0',
      pageno: Math.floor((query.offset || 0) / 10) + 1
    });

    const response = await fetch(`${this.config.endpoint}/search?${params}`, {
      headers: this.config.apiKey ? {
        'Authorization': `Bearer ${this.config.apiKey}`
      } : {}
    });

    if (!response.ok) {
      throw new Error(`SearXNG search failed: ${response.status}`);
    }

    const data = await response.json();

    return {
      results: data.results.map((r: any, i: number) => ({
        title: r.title,
        url: r.url,
        snippet: r.content,
        position: i + 1,
        source: 'searxng',
        metadata: {
          engine: r.engine,
          score: r.score
        }
      })),
      totalResults: data.number_of_results,
      searchTime: data.search.duration,
      sources: ['searxng'],
      query
    };
  }

  private mapType(type?: string): string {
    const typeMap: Record<string, string> = {
      'web': 'general',
      'image': 'images',
      'video': 'videos',
      'news': 'news'
    };
    return typeMap[type || 'web'] || 'general';
  }
}
```

**File**: `apps/api/src/services/search/engines/DuckDuckGoScraper.ts`

```typescript
export class DuckDuckGoScraper implements SearchEngine {
  private scraper: StealthBrowserService;

  constructor() {
    this.scraper = new StealthBrowserService();
  }

  async search(query: SearchQuery): Promise<SearchResponse> {
    const startTime = Date.now();
    const page = await this.scraper.createStealthPage();

    try {
      // Navigate to DuckDuckGo
      await page.goto('https://duckduckgo.com', {
        waitUntil: 'networkidle'
      });

      // Enter search query
      await page.type('input[name="q"]', query.query);
      await page.keyboard.press('Enter');

      // Wait for results
      await page.waitForSelector('.results', { timeout: 10000 });

      // Extract results
      const results = await page.evaluate(() => {
        const items: any[] = [];
        const resultElements = document.querySelectorAll('.result');

        resultElements.forEach((el, index) => {
          const titleEl = el.querySelector('.result__title a');
          const snippetEl = el.querySelector('.result__snippet');

          if (titleEl) {
            items.push({
              title: titleEl.textContent?.trim(),
              url: (titleEl as HTMLAnchorElement).href,
              snippet: snippetEl?.textContent?.trim() || '',
              position: index + 1
            });
          }
        });

        return items;
      });

      return {
        results: results.map(r => ({
          ...r,
          source: 'duckduckgo'
        })),
        searchTime: Date.now() - startTime,
        sources: ['duckduckgo'],
        query
      };
    } finally {
      await page.close();
    }
  }
}
```

**File**: `apps/api/src/services/search/engines/SerperEngine.ts`

```typescript
export class SerperEngine implements SearchEngine {
  private apiKey: string;
  private baseUrl = 'https://google.serper.dev';

  constructor(config: { apiKey: string }) {
    this.apiKey = config.apiKey;
  }

  async search(query: SearchQuery): Promise<SearchResponse> {
    const response = await fetch(`${this.baseUrl}/search`, {
      method: 'POST',
      headers: {
        'X-API-KEY': this.apiKey,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        q: query.query,
        num: query.limit || 10,
        start: query.offset || 0,
        gl: query.country || 'us',
        hl: query.language || 'en',
        type: query.type || 'search',
        tbs: this.buildTimeRange(query.timeRange)
      })
    });

    if (!response.ok) {
      throw new Error(`Serper API error: ${response.status}`);
    }

    const data = await response.json();

    return {
      results: (data.organic || []).map((r: any, i: number) => ({
        title: r.title,
        url: r.link,
        snippet: r.snippet,
        position: r.position || i + 1,
        source: 'serper',
        metadata: {
          date: r.date,
          sitelinks: r.sitelinks
        }
      })),
      totalResults: data.searchInformation?.totalResults,
      searchTime: data.searchInformation?.searchTime,
      sources: ['serper'],
      query
    };
  }

  private buildTimeRange(range?: string): string {
    const ranges: Record<string, string> = {
      'day': 'qdr:d',
      'week': 'qdr:w',
      'month': 'qdr:m',
      'year': 'qdr:y'
    };
    return ranges[range || ''] || '';
  }
}
```

### Phase 2: Result Aggregation & Ranking (Week 1-2)

#### 2.1 Intelligent Result Aggregator

**File**: `apps/api/src/services/search/ResultAggregator.ts`

```typescript
export class ResultAggregator {
  private ranker: ResultRanker;
  private deduplicator: ResultDeduplicator;

  constructor() {
    this.ranker = new ResultRanker();
    this.deduplicator = new ResultDeduplicator();
  }

  async aggregate(results: SearchResult[]): Promise<SearchResult[]> {
    // Step 1: Deduplicate results
    const deduplicated = await this.deduplicator.deduplicate(results);

    // Step 2: Calculate relevance scores
    const scored = await this.ranker.score(deduplicated);

    // Step 3: Sort by score
    scored.sort((a, b) => b.score - a.score);

    // Step 4: Normalize positions
    return scored.map((result, index) => ({
      ...result,
      position: index + 1
    }));
  }
}

export class ResultDeduplicator {
  deduplicate(results: SearchResult[]): SearchResult[] {
    const seen = new Map<string, SearchResult>();
    const urlPatterns = new Map<string, SearchResult>();

    for (const result of results) {
      // Normalize URL
      const normalizedUrl = this.normalizeUrl(result.url);

      // Check exact match
      if (seen.has(normalizedUrl)) {
        // Merge metadata from duplicate
        const existing = seen.get(normalizedUrl)!;
        this.mergeResults(existing, result);
        continue;
      }

      // Check similar URLs
      const similarUrl = this.findSimilarUrl(normalizedUrl, urlPatterns);
      if (similarUrl) {
        const existing = urlPatterns.get(similarUrl)!;
        this.mergeResults(existing, result);
        continue;
      }

      seen.set(normalizedUrl, result);
      urlPatterns.set(normalizedUrl, result);
    }

    return Array.from(seen.values());
  }

  private normalizeUrl(url: string): string {
    try {
      const parsed = new URL(url);
      // Remove common tracking parameters
      ['utm_source', 'utm_medium', 'utm_campaign', 'ref', 'source'].forEach(param => {
        parsed.searchParams.delete(param);
      });
      // Remove trailing slash
      let normalized = parsed.toString();
      if (normalized.endsWith('/')) {
        normalized = normalized.slice(0, -1);
      }
      return normalized.toLowerCase();
    } catch {
      return url.toLowerCase();
    }
  }

  private findSimilarUrl(url: string, patterns: Map<string, SearchResult>): string | null {
    // Check for very similar URLs (e.g., http vs https, www vs non-www)
    for (const [existingUrl] of patterns) {
      if (this.areSimilar(url, existingUrl)) {
        return existingUrl;
      }
    }
    return null;
  }

  private areSimilar(url1: string, url2: string): boolean {
    try {
      const parsed1 = new URL(url1);
      const parsed2 = new URL(url2);

      // Same path and host (ignoring www)
      const host1 = parsed1.hostname.replace('www.', '');
      const host2 = parsed2.hostname.replace('www.', '');

      return host1 === host2 && parsed1.pathname === parsed2.pathname;
    } catch {
      return false;
    }
  }

  private mergeResults(existing: SearchResult, duplicate: SearchResult): void {
    // Merge sources
    if (!existing.metadata) existing.metadata = {};
    if (!existing.metadata.sources) existing.metadata.sources = [existing.source];
    existing.metadata.sources.push(duplicate.source);

    // Use better snippet if available
    if (duplicate.snippet.length > existing.snippet.length) {
      existing.snippet = duplicate.snippet;
    }

    // Average positions
    if (!existing.metadata.positions) {
      existing.metadata.positions = { [existing.source]: existing.position };
    }
    existing.metadata.positions[duplicate.source] = duplicate.position;
  }
}

export class ResultRanker {
  async score(results: SearchResult[]): Promise<ScoredResult[]> {
    return results.map(result => {
      let score = 0;

      // Position score (primary factor)
      score += this.calculatePositionScore(result);

      // Source reliability score
      score += this.calculateSourceScore(result);

      // Content quality score
      score += this.calculateContentScore(result);

      // Multi-source boost
      score += this.calculateMultiSourceBoost(result);

      // Domain authority (simplified)
      score += this.calculateDomainScore(result);

      return {
        ...result,
        score
      } as ScoredResult;
    });
  }

  private calculatePositionScore(result: SearchResult): number {
    // Inverse position scoring
    if (result.metadata?.positions) {
      const positions = Object.values(result.metadata.positions);
      const avgPosition = positions.reduce((a, b) => a + b, 0) / positions.length;
      return 100 / avgPosition;
    }
    return 100 / result.position;
  }

  private calculateSourceScore(result: SearchResult): number {
    const sourceScores: Record<string, number> = {
      'google': 10,
      'serper': 10,
      'serpapi': 9,
      'bing': 8,
      'brave': 8,
      'searxng': 7,
      'duckduckgo': 7,
      'yandex': 6,
      'qwant': 6
    };

    if (result.metadata?.sources) {
      const scores = result.metadata.sources.map(s => sourceScores[s] || 5);
      return scores.reduce((a, b) => a + b, 0) / scores.length;
    }

    return sourceScores[result.source] || 5;
  }

  private calculateContentScore(result: SearchResult): number {
    let score = 0;

    // Title relevance (simplified)
    if (result.title && result.title.length > 10) {
      score += 5;
    }

    // Snippet quality
    if (result.snippet && result.snippet.length > 50) {
      score += 5;
    }

    // Has metadata
    if (result.metadata && Object.keys(result.metadata).length > 0) {
      score += 3;
    }

    return score;
  }

  private calculateMultiSourceBoost(result: SearchResult): number {
    if (result.metadata?.sources && result.metadata.sources.length > 1) {
      return result.metadata.sources.length * 5;
    }
    return 0;
  }

  private calculateDomainScore(result: SearchResult): number {
    try {
      const domain = new URL(result.url).hostname;

      // High-authority domains (simplified)
      const authorityDomains = [
        'wikipedia.org', 'github.com', 'stackoverflow.com',
        'microsoft.com', 'google.com', 'aws.amazon.com',
        'mozilla.org', 'w3.org'
      ];

      if (authorityDomains.some(d => domain.includes(d))) {
        return 10;
      }
    } catch {
      // Invalid URL
    }

    return 0;
  }
}

interface ScoredResult extends SearchResult {
  score: number;
}
```

### Phase 3: Search Caching & Optimization (Week 2)

#### 3.1 Intelligent Cache System

**File**: `apps/api/src/services/search/SearchCache.ts`

```typescript
export class SearchCache {
  private redis: Redis;
  private memoryCache: LRUCache<string, SearchResponse>;

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!);
    this.memoryCache = new LRUCache({
      max: 1000,
      ttl: 1000 * 60 * 5 // 5 minutes
    });
  }

  async get(query: SearchQuery): Promise<SearchResponse | null> {
    const key = this.generateKey(query);

    // Check memory cache first
    const memCached = this.memoryCache.get(key);
    if (memCached) {
      return memCached;
    }

    // Check Redis
    const redisCached = await this.redis.get(key);
    if (redisCached) {
      const response = JSON.parse(redisCached);
      // Populate memory cache
      this.memoryCache.set(key, response);
      return response;
    }

    return null;
  }

  async set(query: SearchQuery, response: SearchResponse): Promise<void> {
    const key = this.generateKey(query);
    const ttl = this.calculateTTL(query);

    // Set in both caches
    this.memoryCache.set(key, response);
    await this.redis.setex(key, ttl, JSON.stringify(response));
  }

  private generateKey(query: SearchQuery): string {
    const normalized = {
      q: query.query.toLowerCase().trim(),
      l: query.limit || 10,
      o: query.offset || 0,
      lang: query.language || 'en',
      country: query.country || 'us',
      time: query.timeRange || 'all',
      type: query.type || 'web'
    };

    return `search:${crypto
      .createHash('sha256')
      .update(JSON.stringify(normalized))
      .digest('hex')}`;
  }

  private calculateTTL(query: SearchQuery): number {
    // Different TTLs based on time range
    const ttlMap: Record<string, number> = {
      'day': 3600,        // 1 hour
      'week': 10800,      // 3 hours
      'month': 21600,     // 6 hours
      'year': 43200,      // 12 hours
      'all': 86400        // 24 hours
    };

    return ttlMap[query.timeRange || 'all'] || 3600;
  }

  async invalidate(pattern: string): Promise<void> {
    const keys = await this.redis.keys(`search:*${pattern}*`);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }

    // Clear memory cache
    this.memoryCache.clear();
  }
}
```

## Monitoring & Analytics

```typescript
interface SearchMetrics {
  totalSearches: number;
  cacheHitRate: number;
  avgSearchTime: number;
  engineUsage: Map<string, number>;
  engineSuccessRate: Map<string, number>;
  topQueries: Array<{ query: string; count: number }>;
  errorRate: number;
}
```

## Configuration

```env
# Primary search configuration
PRIMARY_SEARCH_ENGINE=searxng

# SearXNG (self-hosted)
SEARXNG_ENDPOINT=http://localhost:8888
SEARXNG_API_KEY=optional_api_key

# Commercial APIs
SERPER_API_KEY=your_serper_key
SERPAPI_KEY=your_serpapi_key
BRAVE_SEARCH_API_KEY=your_brave_key
BING_API_KEY=your_bing_key

# Cache settings
SEARCH_CACHE_TTL=3600
SEARCH_CACHE_SIZE=1000

# Rate limits
SEARCH_RATE_LIMIT_PER_MINUTE=60
```

## Docker Compose for SearXNG

```yaml
version: '3.7'

services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    ports:
      - "8888:8080"
    volumes:
      - ./searxng:/etc/searxng
    environment:
      - SEARXNG_BASE_URL=http://localhost:8888
      - SEARXNG_SECRET_KEY=${SEARXNG_SECRET}
    restart: unless-stopped
```

## Implementation Timeline

- **Week 1**: Multi-engine search service
- **Week 2**: Result aggregation and caching
- **Week 3**: Testing and optimization
- **Week 4**: Monitoring and analytics

## Success Criteria

1. **Zero dependency on Google scraping**
2. **<500ms average search time**
3. **>90% cache hit rate for repeated queries**
4. **Result relevance score >85%**
5. **99.9% availability through fallbacks**

## Dependencies

```json
{
  "lru-cache": "^10.4.3",
  "crypto": "built-in"
}
```