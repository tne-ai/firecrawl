# Advanced Proxy Management System Implementation Plan

## Executive Summary
Implement a sophisticated proxy management and rotation system to avoid IP-based detection and blocking. This plan details the integration of residential/mobile proxies, intelligent rotation algorithms, health checking, and geographic targeting capabilities.

## Current State Analysis
- **Location**: Environment variables in `apps/playwright-service-ts/api.ts`
- **Current Implementation**:
  - Basic static proxy support via PROXY_SERVER, PROXY_USERNAME, PROXY_PASSWORD
  - No rotation capability
  - No health checking
  - No fallback mechanism
  - No geographic selection

## Recommended Proxy Provider Stack

### Tier 1: Enterprise Providers (Best Performance)

#### 1. Bright Data (Recommended for Enterprise)
- **Pool Size**: 72+ million residential IPs
- **Pricing**: Starting at $8.40/GB for pay-as-you-go
- **API**: RESTful + SDK support
- **Features**:
  - SERP API included
  - Web Unlocker technology
  - 99.9% uptime SLA

#### 2. Oxylabs
- **Pool Size**: 175+ million IPs (largest globally)
- **Pricing**: Custom enterprise pricing
- **API**: Advanced developer-focused APIs
- **Features**:
  - City/ASN/ZIP targeting
  - Dedicated account manager
  - Real-time performance stats

### Tier 2: Mid-Market Providers

#### 3. Smartproxy (Now Decodo)
- **Pool Size**: 55+ million IPs
- **Pricing**: Starting at $7/GB
- **Features**: 3-day free trial, good for SMBs

#### 4. IPRoyal
- **Pool Size**: 32+ million IPs
- **Pricing**: Non-expiring traffic packages from $7/GB
- **Features**: Budget-friendly, SOCKS5 support

## System Architecture

### Core Components

```typescript
// apps/api/src/services/proxy-manager/types.ts
export interface ProxyProvider {
  name: string;
  type: 'residential' | 'datacenter' | 'mobile';
  endpoint: string;
  auth: {
    username: string;
    password: string;
  };
  features: {
    stickySession: boolean;
    geoTargeting: boolean;
    protocolSupport: ('http' | 'https' | 'socks5')[];
  };
}

export interface ProxyConfig {
  url: string;
  username?: string;
  password?: string;
  location?: string;
  sessionId?: string;
  provider: string;
  type: 'residential' | 'datacenter' | 'mobile';
  lastUsed?: Date;
  failureCount: number;
  successCount: number;
  avgResponseTime: number;
  metadata: Record<string, any>;
}

export interface ProxyRotationStrategy {
  type: 'round-robin' | 'random' | 'weighted' | 'geo-optimized' | 'least-used';
  stickySession: boolean;
  sessionDuration?: number;
  fallbackStrategy?: ProxyRotationStrategy;
}
```

### Phase 1: Proxy Pool Manager (Week 1)

#### 1.1 Core Proxy Manager Service

**File**: `apps/api/src/services/proxy-manager/ProxyManager.ts`

```typescript
import { Redis } from 'ioredis';
import { logger } from '../../lib/logger';

export class ProxyManager {
  private providers: Map<string, ProxyProvider> = new Map();
  private proxyPool: Map<string, ProxyConfig[]> = new Map();
  private healthChecker: ProxyHealthChecker;
  private redis: Redis;
  private rotationStrategy: ProxyRotationStrategy;

  constructor(redis: Redis, strategy: ProxyRotationStrategy) {
    this.redis = redis;
    this.rotationStrategy = strategy;
    this.healthChecker = new ProxyHealthChecker(this);
    this.initializeProviders();
  }

  private async initializeProviders() {
    // Initialize Bright Data
    if (process.env.BRIGHTDATA_USERNAME) {
      this.addProvider({
        name: 'brightdata',
        type: 'residential',
        endpoint: 'http://brd.superproxy.io:22225',
        auth: {
          username: process.env.BRIGHTDATA_USERNAME,
          password: process.env.BRIGHTDATA_PASSWORD
        },
        features: {
          stickySession: true,
          geoTargeting: true,
          protocolSupport: ['http', 'https', 'socks5']
        }
      });
    }

    // Initialize Oxylabs
    if (process.env.OXYLABS_USERNAME) {
      this.addProvider({
        name: 'oxylabs',
        type: 'residential',
        endpoint: 'http://pr.oxylabs.io:7777',
        auth: {
          username: process.env.OXYLABS_USERNAME,
          password: process.env.OXYLABS_PASSWORD
        },
        features: {
          stickySession: true,
          geoTargeting: true,
          protocolSupport: ['http', 'https']
        }
      });
    }

    // Initialize IPRoyal
    if (process.env.IPROYAL_USERNAME) {
      this.addProvider({
        name: 'iproyal',
        type: 'residential',
        endpoint: 'http://geo.iproyal.com:12321',
        auth: {
          username: process.env.IPROYAL_USERNAME,
          password: process.env.IPROYAL_PASSWORD
        },
        features: {
          stickySession: false,
          geoTargeting: true,
          protocolSupport: ['http', 'https', 'socks5']
        }
      });
    }

    await this.loadProxyPool();
  }

  async getProxy(options: {
    targetUrl: string;
    location?: string;
    type?: 'residential' | 'datacenter' | 'mobile';
    sessionId?: string;
    preferredProvider?: string;
  }): Promise<ProxyConfig> {
    const domain = new URL(options.targetUrl).hostname;

    // Check for sticky session
    if (options.sessionId) {
      const sessionProxy = await this.redis.get(`session:${options.sessionId}`);
      if (sessionProxy) {
        return JSON.parse(sessionProxy);
      }
    }

    // Get proxy based on rotation strategy
    let proxy: ProxyConfig;
    switch (this.rotationStrategy.type) {
      case 'geo-optimized':
        proxy = await this.getGeoOptimizedProxy(options);
        break;
      case 'weighted':
        proxy = await this.getWeightedProxy(options);
        break;
      case 'least-used':
        proxy = await this.getLeastUsedProxy(options);
        break;
      case 'round-robin':
        proxy = await this.getRoundRobinProxy(options);
        break;
      default:
        proxy = await this.getRandomProxy(options);
    }

    // Store session if needed
    if (options.sessionId && this.rotationStrategy.stickySession) {
      await this.redis.setex(
        `session:${options.sessionId}`,
        this.rotationStrategy.sessionDuration || 600,
        JSON.stringify(proxy)
      );
    }

    // Track usage
    await this.trackProxyUsage(proxy, domain);

    return proxy;
  }

  private async getGeoOptimizedProxy(options: any): Promise<ProxyConfig> {
    // Get target server location
    const targetLocation = await this.detectTargetLocation(options.targetUrl);

    // Find proxies in the same region
    const regionalProxies = await this.filterProxiesByLocation(
      targetLocation.country,
      targetLocation.region
    );

    if (regionalProxies.length === 0) {
      // Fallback to any available proxy
      return this.getRandomProxy(options);
    }

    // Select best proxy from region based on performance
    return this.selectBestProxy(regionalProxies);
  }

  private async getWeightedProxy(options: any): Promise<ProxyConfig> {
    const availableProxies = await this.getAvailableProxies(options);

    // Calculate weights based on success rate and response time
    const weightedProxies = availableProxies.map(proxy => ({
      proxy,
      weight: this.calculateProxyWeight(proxy)
    }));

    // Weighted random selection
    const totalWeight = weightedProxies.reduce((sum, p) => sum + p.weight, 0);
    let random = Math.random() * totalWeight;

    for (const { proxy, weight } of weightedProxies) {
      random -= weight;
      if (random <= 0) {
        return proxy;
      }
    }

    return weightedProxies[0].proxy;
  }

  private calculateProxyWeight(proxy: ProxyConfig): number {
    const successRate = proxy.successCount / (proxy.successCount + proxy.failureCount || 1);
    const responseTimeScore = 1 / (proxy.avgResponseTime / 1000 || 1); // Inverse of seconds
    return successRate * responseTimeScore * 100;
  }
}
```

#### 1.2 Health Checking System

**File**: `apps/api/src/services/proxy-manager/ProxyHealthChecker.ts`

```typescript
export class ProxyHealthChecker {
  private manager: ProxyManager;
  private checkInterval: number = 60000; // 1 minute
  private unhealthyThreshold: number = 3;
  private healthyThreshold: number = 2;

  constructor(manager: ProxyManager) {
    this.manager = manager;
    this.startHealthChecks();
  }

  private async startHealthChecks() {
    setInterval(async () => {
      await this.performHealthChecks();
    }, this.checkInterval);
  }

  private async performHealthChecks() {
    const proxies = await this.manager.getAllProxies();
    const checkPromises = proxies.map(proxy => this.checkProxy(proxy));

    const results = await Promise.allSettled(checkPromises);

    for (let i = 0; i < results.length; i++) {
      const result = results[i];
      const proxy = proxies[i];

      if (result.status === 'fulfilled' && result.value.healthy) {
        await this.markHealthy(proxy);
      } else {
        await this.markUnhealthy(proxy, result.reason);
      }
    }
  }

  private async checkProxy(proxy: ProxyConfig): Promise<{ healthy: boolean; latency?: number }> {
    const startTime = Date.now();

    try {
      const response = await fetch('https://httpbin.org/ip', {
        agent: new HttpsProxyAgent(proxy.url),
        timeout: 10000,
        signal: AbortSignal.timeout(10000)
      });

      if (!response.ok) {
        return { healthy: false };
      }

      const data = await response.json();
      const latency = Date.now() - startTime;

      // Verify proxy is actually working
      if (!data.origin || data.origin === await this.getLocalIP()) {
        return { healthy: false };
      }

      return { healthy: true, latency };
    } catch (error) {
      logger.error('Proxy health check failed', { proxy: proxy.url, error });
      return { healthy: false };
    }
  }

  private async markHealthy(proxy: ProxyConfig) {
    const key = `proxy:health:${this.getProxyId(proxy)}`;
    const current = await this.manager.redis.get(key);
    const health = current ? JSON.parse(current) : { consecutive: 0, status: 'unknown' };

    health.consecutive = health.consecutive < 0 ? 1 : health.consecutive + 1;

    if (health.consecutive >= this.healthyThreshold) {
      health.status = 'healthy';
      await this.manager.enableProxy(proxy);
    }

    await this.manager.redis.setex(key, 3600, JSON.stringify(health));
  }

  private async markUnhealthy(proxy: ProxyConfig, reason?: any) {
    const key = `proxy:health:${this.getProxyId(proxy)}`;
    const current = await this.manager.redis.get(key);
    const health = current ? JSON.parse(current) : { consecutive: 0, status: 'unknown' };

    health.consecutive = health.consecutive > 0 ? -1 : health.consecutive - 1;
    health.lastError = reason;
    health.lastErrorTime = new Date().toISOString();

    if (Math.abs(health.consecutive) >= this.unhealthyThreshold) {
      health.status = 'unhealthy';
      await this.manager.disableProxy(proxy);
    }

    await this.manager.redis.setex(key, 3600, JSON.stringify(health));
  }
}
```

### Phase 2: Provider Integration (Week 1-2)

#### 2.1 Bright Data Integration

**File**: `apps/api/src/services/proxy-manager/providers/BrightDataProvider.ts`

```typescript
export class BrightDataProvider implements ProxyProvider {
  private baseUrl = 'https://api.brightdata.com';

  async getProxy(options: {
    country?: string;
    city?: string;
    state?: string;
    asn?: number;
    carrier?: string;
    sessionId?: string;
  }): Promise<ProxyConfig> {
    const username = this.buildUsername(options);

    return {
      url: `http://${username}:${this.password}@brd.superproxy.io:22225`,
      username,
      password: this.password,
      location: options.country,
      sessionId: options.sessionId,
      provider: 'brightdata',
      type: 'residential',
      failureCount: 0,
      successCount: 0,
      avgResponseTime: 0,
      metadata: {
        country: options.country,
        city: options.city,
        asn: options.asn
      }
    };
  }

  private buildUsername(options: any): string {
    const parts = [this.baseUsername];

    if (options.sessionId) {
      parts.push(`session-${options.sessionId}`);
    }

    if (options.country) {
      parts.push(`country-${options.country}`);
    }

    if (options.city) {
      parts.push(`city-${options.city.replace(/ /g, '_')}`);
    }

    if (options.state) {
      parts.push(`state-${options.state}`);
    }

    if (options.asn) {
      parts.push(`asn-${options.asn}`);
    }

    return parts.join('-');
  }

  async getAvailableCountries(): Promise<string[]> {
    const response = await fetch(`${this.baseUrl}/zones/countries`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });

    return response.json();
  }

  async getUsageStats(): Promise<{
    bandwidth: number;
    requests: number;
    cost: number;
  }> {
    const response = await fetch(`${this.baseUrl}/stats/usage`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });

    return response.json();
  }
}
```

#### 2.2 Dynamic Provider Selection

**File**: `apps/api/src/services/proxy-manager/ProviderSelector.ts`

```typescript
export class ProviderSelector {
  private providers: Map<string, ProxyProvider> = new Map();
  private providerStats: Map<string, ProviderStats> = new Map();

  async selectBestProvider(requirements: {
    targetDomain: string;
    requiredLocation?: string;
    requiredType?: 'residential' | 'datacenter' | 'mobile';
    budget?: number;
  }): Promise<string> {
    // Get historical performance for this domain
    const domainStats = await this.getDomainStats(requirements.targetDomain);

    // Filter providers by requirements
    const eligibleProviders = await this.filterProviders(requirements);

    // Score each provider
    const scores = eligibleProviders.map(provider => ({
      provider,
      score: this.calculateProviderScore(provider, requirements, domainStats)
    }));

    // Sort by score
    scores.sort((a, b) => b.score - a.score);

    if (scores.length === 0) {
      throw new Error('No suitable proxy providers available');
    }

    return scores[0].provider;
  }

  private calculateProviderScore(
    provider: string,
    requirements: any,
    domainStats: any
  ): number {
    let score = 100;
    const stats = this.providerStats.get(provider);

    if (!stats) return 50; // Default score for new provider

    // Success rate weight: 40%
    score *= stats.successRate * 0.4;

    // Speed weight: 30%
    const speedScore = Math.min(1, 1000 / stats.avgResponseTime);
    score += speedScore * 30;

    // Cost weight: 20%
    if (requirements.budget) {
      const costScore = Math.min(1, requirements.budget / stats.costPerGB);
      score += costScore * 20;
    }

    // Domain-specific performance: 10%
    if (domainStats[provider]) {
      score += domainStats[provider].successRate * 10;
    }

    return score;
  }

  async monitorProviderPerformance(
    provider: string,
    result: {
      success: boolean;
      responseTime: number;
      bytesTransferred: number;
    }
  ) {
    const stats = this.providerStats.get(provider) || {
      totalRequests: 0,
      successfulRequests: 0,
      totalResponseTime: 0,
      totalBytes: 0,
      successRate: 1,
      avgResponseTime: 0,
      costPerGB: 0
    };

    stats.totalRequests++;
    stats.totalResponseTime += result.responseTime;
    stats.totalBytes += result.bytesTransferred;

    if (result.success) {
      stats.successfulRequests++;
    }

    stats.successRate = stats.successfulRequests / stats.totalRequests;
    stats.avgResponseTime = stats.totalResponseTime / stats.totalRequests;

    this.providerStats.set(provider, stats);

    // Persist to Redis
    await this.redis.setex(
      `provider:stats:${provider}`,
      3600,
      JSON.stringify(stats)
    );
  }
}
```

### Phase 3: Integration with Scraper Engines (Week 2)

#### 3.1 Enhanced Engine Integration

**File**: `apps/api/src/scraper/scrapeURL/engines/utils/proxyMiddleware.ts`

```typescript
export class ProxyMiddleware {
  private proxyManager: ProxyManager;

  async configureRequest(
    meta: Meta,
    requestOptions: any
  ): Promise<any> {
    // Determine proxy requirements
    const proxyOptions = {
      targetUrl: meta.url,
      location: meta.options.location,
      type: this.getProxyType(meta),
      sessionId: this.generateSessionId(meta),
      preferredProvider: meta.options.proxyProvider
    };

    // Get appropriate proxy
    const proxy = await this.proxyManager.getProxy(proxyOptions);

    // Configure request with proxy
    if (proxy.type === 'socks5') {
      requestOptions.agent = new SocksProxyAgent(proxy.url);
    } else {
      requestOptions.agent = new HttpsProxyAgent(proxy.url);
    }

    // Add proxy authentication headers if needed
    if (proxy.username && proxy.password) {
      const auth = Buffer.from(`${proxy.username}:${proxy.password}`).toString('base64');
      requestOptions.headers = {
        ...requestOptions.headers,
        'Proxy-Authorization': `Basic ${auth}`
      };
    }

    // Store proxy info in meta for tracking
    meta.proxyUsed = proxy;

    return requestOptions;
  }

  private getProxyType(meta: Meta): 'residential' | 'datacenter' | 'mobile' {
    // Use mobile for mobile user agents
    if (meta.options.mobile) {
      return 'mobile';
    }

    // Use residential for high-security targets
    if (this.isHighSecurityTarget(meta.url)) {
      return 'residential';
    }

    // Default to datacenter for speed
    return 'datacenter';
  }

  private isHighSecurityTarget(url: string): boolean {
    const highSecurityDomains = [
      'amazon.com',
      'ebay.com',
      'linkedin.com',
      'instagram.com',
      'twitter.com',
      'facebook.com'
    ];

    const domain = new URL(url).hostname;
    return highSecurityDomains.some(d => domain.includes(d));
  }

  private generateSessionId(meta: Meta): string {
    // Generate consistent session ID for sticky sessions
    if (meta.options.maintainSession) {
      return `${meta.id}-${new URL(meta.url).hostname}`;
    }
    return null;
  }

  async handleProxyFailure(
    meta: Meta,
    error: any,
    proxy: ProxyConfig
  ): Promise<void> {
    // Record failure
    await this.proxyManager.recordFailure(proxy, error);

    // Check if we should retry with different proxy
    if (this.shouldRetryWithDifferentProxy(error)) {
      // Blacklist current proxy temporarily
      await this.proxyManager.blacklistProxy(proxy, 300); // 5 minutes

      // Get new proxy
      const newProxy = await this.proxyManager.getProxy({
        targetUrl: meta.url,
        excludeProxy: proxy.url
      });

      meta.proxyUsed = newProxy;
    }
  }

  private shouldRetryWithDifferentProxy(error: any): boolean {
    const retryableErrors = [
      'ECONNREFUSED',
      'ETIMEDOUT',
      'ECONNRESET',
      'PROXY_AUTH_FAILED',
      '407' // Proxy Authentication Required
    ];

    return retryableErrors.some(e =>
      error.message?.includes(e) ||
      error.code === e ||
      error.statusCode === parseInt(e)
    );
  }
}
```

## Testing & Monitoring

### Performance Metrics
```typescript
interface ProxyMetrics {
  provider: string;
  successRate: number;
  avgResponseTime: number;
  costPerRequest: number;
  geoDistribution: Map<string, number>;
  failureReasons: Map<string, number>;
  captchaRate: number;
  blockRate: number;
}
```

### Monitoring Dashboard
- Real-time proxy health status
- Provider performance comparison
- Cost tracking per provider
- Geographic coverage heatmap
- Domain-specific success rates

## Cost Optimization

### Smart Provider Selection
1. Use datacenter proxies for low-risk targets
2. Reserve residential for high-security sites
3. Implement cost caps per domain
4. Cache successful proxy-domain pairs

### Budget Management
```typescript
class ProxyBudgetManager {
  async shouldUseProxy(
    domain: string,
    proxyType: string
  ): Promise<boolean> {
    const dailySpend = await this.getDailySpend(domain);
    const budgetLimit = this.getBudgetLimit(domain);

    if (dailySpend >= budgetLimit) {
      // Fallback to cheaper option or direct connection
      return false;
    }

    return true;
  }
}
```

## Implementation Timeline

- **Week 1**: Core proxy manager and health checking
- **Week 2**: Provider integrations (Bright Data, Oxylabs, IPRoyal)
- **Week 3**: Smart rotation algorithms and geo-optimization
- **Week 4**: Testing and monitoring setup

## Success Metrics

1. 90% reduction in IP-based blocks
2. <5% proxy failure rate
3. Average proxy response time <2 seconds
4. Cost per successful request <$0.01
5. Geographic coverage: 150+ countries

## Dependencies

```json
{
  "https-proxy-agent": "^7.0.5",
  "socks-proxy-agent": "^8.0.4",
  "proxy-chain": "^2.5.3",
  "geoip-lite": "^1.4.10",
  "p-queue": "^8.0.1",
  "node-fetch": "^3.3.2"
}
```