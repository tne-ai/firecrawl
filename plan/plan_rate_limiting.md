# Intelligent Rate Limiting & Throttling Implementation Plan

## Executive Summary
Implement an adaptive rate limiting system with intelligent throttling, exponential backoff, circuit breakers, and domain-specific strategies. This plan transforms the basic Redis rate limiter into a sophisticated traffic management system that maximizes throughput while avoiding blocks.

## Current State Analysis
- **Location**: `apps/api/src/services/rate-limiter.ts`
- **Current Implementation**:
  - Basic Redis rate limiting with fixed limits
  - No adaptive throttling
  - No domain-specific rules
  - No exponential backoff
  - No circuit breaker pattern

## Core Concepts

### Rate Limiting Strategies
1. **Token Bucket**: Fixed rate with burst capacity
2. **Sliding Window**: Smooth rate distribution
3. **Adaptive Throttling**: Dynamic adjustment based on response
4. **Circuit Breaker**: Fail-fast for overloaded services
5. **Exponential Backoff**: Progressive delay on failures

## Implementation Architecture

### Phase 1: Enhanced Rate Limiter (Week 1)

#### 1.1 Adaptive Rate Limiter Core

**File**: `apps/api/src/services/rate-limiting/AdaptiveRateLimiter.ts`

```typescript
import { Redis } from 'ioredis';
import { EventEmitter } from 'events';

export interface RateLimitConfig {
  type: 'token-bucket' | 'sliding-window' | 'adaptive';
  limits: {
    perSecond?: number;
    perMinute?: number;
    perHour?: number;
    perDay?: number;
    burst?: number;
  };
  adaptiveConfig?: {
    targetSuccessRate: number;
    adjustmentFactor: number;
    minRate: number;
    maxRate: number;
    windowSize: number;
  };
}

export interface DomainConfig {
  domain: string;
  pattern?: string;
  rateLimit: RateLimitConfig;
  respectRobotsTxt: boolean;
  crawlDelay?: number;
  backoffStrategy?: BackoffStrategy;
  circuitBreaker?: CircuitBreakerConfig;
}

export class AdaptiveRateLimiter extends EventEmitter {
  private redis: Redis;
  private domainConfigs: Map<string, DomainConfig> = new Map();
  private metrics: Map<string, DomainMetrics> = new Map();
  private circuitBreakers: Map<string, CircuitBreaker> = new Map();

  constructor(redis: Redis) {
    super();
    this.redis = redis;
    this.loadDomainConfigs();
    this.startMetricsCollection();
  }

  async checkLimit(url: string, weight: number = 1): Promise<RateLimitResult> {
    const domain = new URL(url).hostname;
    const config = this.getDomainConfig(domain);

    // Check circuit breaker first
    const breaker = this.getCircuitBreaker(domain);
    if (breaker && breaker.state === 'open') {
      return {
        allowed: false,
        retryAfter: breaker.nextAttempt,
        reason: 'circuit_breaker_open'
      };
    }

    // Apply rate limiting based on strategy
    let result: RateLimitResult;
    switch (config.rateLimit.type) {
      case 'token-bucket':
        result = await this.tokenBucketCheck(domain, config, weight);
        break;
      case 'sliding-window':
        result = await this.slidingWindowCheck(domain, config, weight);
        break;
      case 'adaptive':
        result = await this.adaptiveCheck(domain, config, weight);
        break;
      default:
        result = await this.tokenBucketCheck(domain, config, weight);
    }

    // Record metrics
    this.recordMetrics(domain, result.allowed);

    // Emit events for monitoring
    this.emit('rate-limit-check', {
      domain,
      allowed: result.allowed,
      strategy: config.rateLimit.type,
      currentRate: this.getCurrentRate(domain)
    });

    return result;
  }

  private async tokenBucketCheck(
    domain: string,
    config: DomainConfig,
    weight: number
  ): Promise<RateLimitResult> {
    const key = `rate-limit:token:${domain}`;
    const now = Date.now();
    const limits = config.rateLimit.limits;

    // Multi-level rate limiting
    const checks = [];
    if (limits.perSecond) {
      checks.push(this.checkTokenBucket(
        `${key}:second`,
        limits.perSecond,
        1000,
        limits.burst || limits.perSecond,
        weight
      ));
    }
    if (limits.perMinute) {
      checks.push(this.checkTokenBucket(
        `${key}:minute`,
        limits.perMinute,
        60000,
        limits.burst || limits.perMinute,
        weight
      ));
    }
    if (limits.perHour) {
      checks.push(this.checkTokenBucket(
        `${key}:hour`,
        limits.perHour,
        3600000,
        limits.burst || limits.perHour,
        weight
      ));
    }

    const results = await Promise.all(checks);
    const blocked = results.find(r => !r.allowed);

    if (blocked) {
      return blocked;
    }

    return { allowed: true, remaining: Math.min(...results.map(r => r.remaining || 0)) };
  }

  private async checkTokenBucket(
    key: string,
    rate: number,
    interval: number,
    burst: number,
    weight: number
  ): Promise<RateLimitResult> {
    const lua = `
      local key = KEYS[1]
      local rate = tonumber(ARGV[1])
      local interval = tonumber(ARGV[2])
      local burst = tonumber(ARGV[3])
      local weight = tonumber(ARGV[4])
      local now = tonumber(ARGV[5])

      local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
      local tokens = tonumber(bucket[1]) or burst
      local last_refill = tonumber(bucket[2]) or now

      -- Refill tokens
      local time_passed = now - last_refill
      local tokens_to_add = (time_passed / interval) * rate
      tokens = math.min(burst, tokens + tokens_to_add)

      if tokens >= weight then
        tokens = tokens - weight
        redis.call('HSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, math.ceil(interval / 1000))
        return {1, tokens}
      else
        local wait_time = ((weight - tokens) / rate) * interval
        return {0, tokens, wait_time}
      end
    `;

    const result = await this.redis.eval(
      lua,
      1,
      key,
      rate.toString(),
      interval.toString(),
      burst.toString(),
      weight.toString(),
      Date.now().toString()
    ) as [number, number, number?];

    if (result[0] === 1) {
      return { allowed: true, remaining: result[1] };
    } else {
      return {
        allowed: false,
        remaining: result[1],
        retryAfter: Date.now() + (result[2] || 0),
        reason: 'rate_limit_exceeded'
      };
    }
  }

  private async slidingWindowCheck(
    domain: string,
    config: DomainConfig,
    weight: number
  ): Promise<RateLimitResult> {
    const key = `rate-limit:sliding:${domain}`;
    const now = Date.now();
    const window = 60000; // 1 minute window
    const limit = config.rateLimit.limits.perMinute || 60;

    const lua = `
      local key = KEYS[1]
      local now = tonumber(ARGV[1])
      local window = tonumber(ARGV[2])
      local limit = tonumber(ARGV[3])
      local weight = tonumber(ARGV[4])

      -- Remove old entries
      redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

      -- Count current requests in window
      local current = redis.call('ZCARD', key)

      if current + weight <= limit then
        -- Add new request with timestamp
        for i = 1, weight do
          redis.call('ZADD', key, now, now .. ':' .. i)
        end
        redis.call('EXPIRE', key, math.ceil(window / 1000))
        return {1, limit - current - weight}
      else
        -- Calculate when the oldest request will expire
        local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
        local retry_after = 0
        if oldest[2] then
          retry_after = tonumber(oldest[2]) + window - now
        end
        return {0, limit - current, retry_after}
      end
    `;

    const result = await this.redis.eval(
      lua,
      1,
      key,
      now.toString(),
      window.toString(),
      limit.toString(),
      weight.toString()
    ) as [number, number, number?];

    if (result[0] === 1) {
      return { allowed: true, remaining: result[1] };
    } else {
      return {
        allowed: false,
        remaining: result[1],
        retryAfter: now + (result[2] || 0),
        reason: 'sliding_window_limit'
      };
    }
  }

  private async adaptiveCheck(
    domain: string,
    config: DomainConfig,
    weight: number
  ): Promise<RateLimitResult> {
    const adaptiveConfig = config.rateLimit.adaptiveConfig!;
    const metrics = this.getMetrics(domain);

    // Calculate current rate based on success rate
    let currentRate = this.getCurrentAdaptiveRate(domain);

    if (metrics.successRate < adaptiveConfig.targetSuccessRate) {
      // Decrease rate if success rate is low
      currentRate *= (1 - adaptiveConfig.adjustmentFactor);
    } else if (metrics.successRate > adaptiveConfig.targetSuccessRate + 0.05) {
      // Increase rate if success rate is high
      currentRate *= (1 + adaptiveConfig.adjustmentFactor);
    }

    // Apply bounds
    currentRate = Math.max(
      adaptiveConfig.minRate,
      Math.min(adaptiveConfig.maxRate, currentRate)
    );

    // Store new rate
    await this.redis.setex(
      `adaptive-rate:${domain}`,
      300,
      currentRate.toString()
    );

    // Use token bucket with adaptive rate
    const adaptedConfig = {
      ...config,
      rateLimit: {
        ...config.rateLimit,
        limits: {
          perMinute: currentRate * 60
        }
      }
    };

    return this.tokenBucketCheck(domain, adaptedConfig, weight);
  }

  private getCurrentAdaptiveRate(domain: string): number {
    // Get current rate from Redis or use default
    const stored = this.redis.get(`adaptive-rate:${domain}`);
    return stored ? parseFloat(stored.toString()) : 1;
  }

  private getMetrics(domain: string): DomainMetrics {
    if (!this.metrics.has(domain)) {
      this.metrics.set(domain, {
        requests: 0,
        successes: 0,
        failures: 0,
        successRate: 1,
        avgResponseTime: 0,
        lastUpdated: Date.now()
      });
    }
    return this.metrics.get(domain)!;
  }

  private recordMetrics(domain: string, success: boolean) {
    const metrics = this.getMetrics(domain);
    metrics.requests++;

    if (success) {
      metrics.successes++;
    } else {
      metrics.failures++;
    }

    metrics.successRate = metrics.successes / metrics.requests;
    metrics.lastUpdated = Date.now();
  }
}
```

### Phase 2: Exponential Backoff & Circuit Breaker (Week 1-2)

#### 2.1 Exponential Backoff Implementation

**File**: `apps/api/src/services/rate-limiting/ExponentialBackoff.ts`

```typescript
export interface BackoffStrategy {
  type: 'exponential' | 'linear' | 'fibonacci' | 'decorrelated-jitter';
  initialDelay: number;
  maxDelay: number;
  multiplier: number;
  jitter: boolean;
  maxRetries: number;
}

export class ExponentialBackoff {
  private attempts: Map<string, number> = new Map();
  private nextRetry: Map<string, number> = new Map();

  constructor(private strategy: BackoffStrategy) {}

  shouldRetry(key: string): boolean {
    const attemptCount = this.attempts.get(key) || 0;

    if (attemptCount >= this.strategy.maxRetries) {
      return false;
    }

    const nextRetryTime = this.nextRetry.get(key) || 0;
    return Date.now() >= nextRetryTime;
  }

  calculateDelay(key: string): number {
    const attemptCount = this.attempts.get(key) || 0;
    this.attempts.set(key, attemptCount + 1);

    let delay: number;

    switch (this.strategy.type) {
      case 'exponential':
        delay = this.exponentialDelay(attemptCount);
        break;
      case 'linear':
        delay = this.linearDelay(attemptCount);
        break;
      case 'fibonacci':
        delay = this.fibonacciDelay(attemptCount);
        break;
      case 'decorrelated-jitter':
        delay = this.decorrelatedJitterDelay(attemptCount);
        break;
      default:
        delay = this.exponentialDelay(attemptCount);
    }

    // Apply jitter if enabled
    if (this.strategy.jitter) {
      delay = this.applyJitter(delay);
    }

    // Cap at max delay
    delay = Math.min(delay, this.strategy.maxDelay);

    // Set next retry time
    this.nextRetry.set(key, Date.now() + delay);

    return delay;
  }

  private exponentialDelay(attempt: number): number {
    return this.strategy.initialDelay * Math.pow(this.strategy.multiplier, attempt);
  }

  private linearDelay(attempt: number): number {
    return this.strategy.initialDelay * (attempt + 1);
  }

  private fibonacciDelay(attempt: number): number {
    if (attempt === 0) return this.strategy.initialDelay;
    if (attempt === 1) return this.strategy.initialDelay * 2;

    let a = this.strategy.initialDelay;
    let b = this.strategy.initialDelay * 2;

    for (let i = 2; i <= attempt; i++) {
      const temp = a + b;
      a = b;
      b = temp;
    }

    return b;
  }

  private decorrelatedJitterDelay(attempt: number): number {
    const prevDelay = this.getPreviousDelay(attempt);
    return Math.random() * Math.min(
      this.strategy.maxDelay,
      prevDelay * 3
    );
  }

  private applyJitter(delay: number): number {
    // Add random jitter between -25% to +25%
    const jitterRange = delay * 0.25;
    return delay + (Math.random() * 2 - 1) * jitterRange;
  }

  private getPreviousDelay(attempt: number): number {
    if (attempt === 0) return this.strategy.initialDelay;
    return this.exponentialDelay(attempt - 1);
  }

  reset(key: string): void {
    this.attempts.delete(key);
    this.nextRetry.delete(key);
  }

  getStatus(key: string): BackoffStatus {
    return {
      attempts: this.attempts.get(key) || 0,
      nextRetryTime: this.nextRetry.get(key),
      canRetry: this.shouldRetry(key)
    };
  }
}

interface BackoffStatus {
  attempts: number;
  nextRetryTime?: number;
  canRetry: boolean;
}
```

#### 2.2 Circuit Breaker Pattern

**File**: `apps/api/src/services/rate-limiting/CircuitBreaker.ts`

```typescript
export interface CircuitBreakerConfig {
  failureThreshold: number;
  successThreshold: number;
  timeout: number;
  halfOpenRequests: number;
  volumeThreshold: number;
  errorThresholdPercentage: number;
  sleepWindow: number;
}

export class CircuitBreaker extends EventEmitter {
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private failures: number = 0;
  private successes: number = 0;
  private lastFailureTime?: number;
  private nextAttempt?: number;
  private halfOpenTests: number = 0;
  private requestCounts: CircularBuffer;

  constructor(
    private name: string,
    private config: CircuitBreakerConfig
  ) {
    super();
    this.requestCounts = new CircularBuffer(this.config.volumeThreshold);
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // Check if circuit breaker should trip
    if (this.state === 'open') {
      if (Date.now() < this.nextAttempt!) {
        throw new CircuitBreakerOpenError(this.name, this.nextAttempt!);
      } else {
        // Transition to half-open
        this.transition('half-open');
      }
    }

    if (this.state === 'half-open' && this.halfOpenTests >= this.config.halfOpenRequests) {
      throw new CircuitBreakerOpenError(this.name, this.nextAttempt!);
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure(error);
      throw error;
    }
  }

  private onSuccess(): void {
    this.requestCounts.add({ success: true, timestamp: Date.now() });

    switch (this.state) {
      case 'half-open':
        this.successes++;
        this.halfOpenTests++;

        if (this.successes >= this.config.successThreshold) {
          this.transition('closed');
        }
        break;

      case 'closed':
        this.failures = 0;
        break;
    }

    this.emit('success', {
      circuit: this.name,
      state: this.state
    });
  }

  private onFailure(error: any): void {
    this.requestCounts.add({ success: false, timestamp: Date.now() });
    this.lastFailureTime = Date.now();

    switch (this.state) {
      case 'half-open':
        this.transition('open');
        break;

      case 'closed':
        this.failures++;

        // Check if we should open the circuit
        if (this.shouldTrip()) {
          this.transition('open');
        }
        break;
    }

    this.emit('failure', {
      circuit: this.name,
      state: this.state,
      error
    });
  }

  private shouldTrip(): boolean {
    // Check volume threshold
    if (this.requestCounts.size() < this.config.volumeThreshold) {
      return false;
    }

    // Check error percentage
    const errorRate = this.requestCounts.errorRate();
    if (errorRate >= this.config.errorThresholdPercentage / 100) {
      return true;
    }

    // Check consecutive failures
    return this.failures >= this.config.failureThreshold;
  }

  private transition(newState: CircuitBreaker['state']): void {
    const oldState = this.state;
    this.state = newState;

    switch (newState) {
      case 'open':
        this.nextAttempt = Date.now() + this.config.sleepWindow;
        this.halfOpenTests = 0;
        break;

      case 'half-open':
        this.successes = 0;
        this.failures = 0;
        this.halfOpenTests = 0;
        break;

      case 'closed':
        this.failures = 0;
        this.successes = 0;
        this.nextAttempt = undefined;
        break;
    }

    this.emit('state-change', {
      circuit: this.name,
      from: oldState,
      to: newState
    });
  }

  getState(): CircuitBreakerState {
    return {
      state: this.state,
      failures: this.failures,
      successes: this.successes,
      nextAttempt: this.nextAttempt,
      errorRate: this.requestCounts.errorRate(),
      totalRequests: this.requestCounts.size()
    };
  }
}

class CircularBuffer {
  private buffer: Array<{ success: boolean; timestamp: number }> = [];
  private pointer: number = 0;

  constructor(private capacity: number) {}

  add(item: { success: boolean; timestamp: number }): void {
    if (this.buffer.length < this.capacity) {
      this.buffer.push(item);
    } else {
      this.buffer[this.pointer] = item;
      this.pointer = (this.pointer + 1) % this.capacity;
    }
  }

  size(): number {
    return this.buffer.length;
  }

  errorRate(): number {
    if (this.buffer.length === 0) return 0;

    const failures = this.buffer.filter(item => !item.success).length;
    return failures / this.buffer.length;
  }
}

export class CircuitBreakerOpenError extends Error {
  constructor(public circuit: string, public retryAfter: number) {
    super(`Circuit breaker ${circuit} is open`);
    this.name = 'CircuitBreakerOpenError';
  }
}
```

### Phase 3: Domain-Specific Configuration (Week 2)

#### 3.1 Domain Configuration Manager

**File**: `apps/api/src/services/rate-limiting/DomainConfigManager.ts`

```typescript
export class DomainConfigManager {
  private configs: Map<string, DomainConfig> = new Map();
  private patterns: Array<{ pattern: RegExp; config: DomainConfig }> = [];

  constructor() {
    this.loadDefaultConfigs();
    this.loadCustomConfigs();
  }

  private loadDefaultConfigs(): void {
    // High-security sites with strict limits
    const highSecurity = {
      rateLimit: {
        type: 'adaptive' as const,
        limits: {
          perSecond: 1,
          perMinute: 30,
          perHour: 500,
          burst: 5
        },
        adaptiveConfig: {
          targetSuccessRate: 0.95,
          adjustmentFactor: 0.1,
          minRate: 0.5,
          maxRate: 2,
          windowSize: 60000
        }
      },
      respectRobotsTxt: true,
      crawlDelay: 2000,
      backoffStrategy: {
        type: 'exponential' as const,
        initialDelay: 1000,
        maxDelay: 60000,
        multiplier: 2,
        jitter: true,
        maxRetries: 5
      },
      circuitBreaker: {
        failureThreshold: 5,
        successThreshold: 3,
        timeout: 30000,
        halfOpenRequests: 3,
        volumeThreshold: 20,
        errorThresholdPercentage: 50,
        sleepWindow: 60000
      }
    };

    // Apply to known high-security domains
    ['amazon.com', 'linkedin.com', 'facebook.com', 'instagram.com'].forEach(domain => {
      this.configs.set(domain, {
        domain,
        ...highSecurity
      });
    });

    // Medium-security sites
    const mediumSecurity = {
      rateLimit: {
        type: 'sliding-window' as const,
        limits: {
          perSecond: 2,
          perMinute: 60,
          perHour: 1000,
          burst: 10
        }
      },
      respectRobotsTxt: true,
      crawlDelay: 1000,
      backoffStrategy: {
        type: 'exponential' as const,
        initialDelay: 500,
        maxDelay: 30000,
        multiplier: 1.5,
        jitter: true,
        maxRetries: 3
      }
    };

    // Default configuration for unknown domains
    this.configs.set('*', {
      domain: '*',
      ...mediumSecurity
    });
  }

  private loadCustomConfigs(): void {
    // Load from environment or config file
    const customConfig = process.env.RATE_LIMIT_CONFIG;
    if (customConfig) {
      try {
        const configs = JSON.parse(customConfig);
        configs.forEach((config: DomainConfig) => {
          if (config.pattern) {
            this.patterns.push({
              pattern: new RegExp(config.pattern),
              config
            });
          } else {
            this.configs.set(config.domain, config);
          }
        });
      } catch (e) {
        console.error('Failed to load custom rate limit config', e);
      }
    }
  }

  getConfig(domain: string): DomainConfig {
    // Check exact match
    if (this.configs.has(domain)) {
      return this.configs.get(domain)!;
    }

    // Check patterns
    for (const { pattern, config } of this.patterns) {
      if (pattern.test(domain)) {
        return config;
      }
    }

    // Check parent domain
    const parts = domain.split('.');
    for (let i = 1; i < parts.length - 1; i++) {
      const parent = parts.slice(i).join('.');
      if (this.configs.has(parent)) {
        return this.configs.get(parent)!;
      }
    }

    // Return default
    return this.configs.get('*')!;
  }

  async updateConfigFromRobotsTxt(domain: string, robotsTxt: string): Promise<void> {
    const crawlDelay = this.parseCrawlDelay(robotsTxt);

    if (crawlDelay) {
      const config = this.getConfig(domain);
      config.crawlDelay = crawlDelay * 1000; // Convert to milliseconds

      // Adjust rate limits based on crawl delay
      if (config.rateLimit.limits.perSecond) {
        config.rateLimit.limits.perSecond = Math.min(
          config.rateLimit.limits.perSecond,
          1 / crawlDelay
        );
      }

      this.configs.set(domain, config);
    }
  }

  private parseCrawlDelay(robotsTxt: string): number | null {
    const match = robotsTxt.match(/Crawl-delay:\s*(\d+)/i);
    return match ? parseInt(match[1]) : null;
  }
}
```

### Phase 4: Integration & Monitoring (Week 2)

#### 4.1 Request Queue with Rate Limiting

**File**: `apps/api/src/services/rate-limiting/RequestQueue.ts`

```typescript
import PQueue from 'p-queue';

export class RateLimitedRequestQueue {
  private queues: Map<string, PQueue> = new Map();
  private rateLimiter: AdaptiveRateLimiter;
  private backoffManager: Map<string, ExponentialBackoff> = new Map();

  constructor(rateLimiter: AdaptiveRateLimiter) {
    this.rateLimiter = rateLimiter;
  }

  async add<T>(
    url: string,
    fn: () => Promise<T>,
    options: {
      priority?: number;
      weight?: number;
      retryOnRateLimit?: boolean;
    } = {}
  ): Promise<T> {
    const domain = new URL(url).hostname;
    const queue = this.getQueue(domain);

    return queue.add(async () => {
      // Check rate limit
      const limitResult = await this.rateLimiter.checkLimit(url, options.weight);

      if (!limitResult.allowed) {
        if (options.retryOnRateLimit) {
          // Wait and retry
          const backoff = this.getBackoff(domain);
          const delay = backoff.calculateDelay(url);

          await new Promise(resolve => setTimeout(resolve, delay));
          return this.add(url, fn, options);
        } else {
          throw new RateLimitError(limitResult);
        }
      }

      try {
        const result = await fn();

        // Reset backoff on success
        const backoff = this.getBackoff(domain);
        backoff.reset(url);

        return result;
      } catch (error) {
        // Handle specific error types
        if (this.isRateLimitError(error)) {
          const backoff = this.getBackoff(domain);
          const delay = backoff.calculateDelay(url);

          if (options.retryOnRateLimit && backoff.shouldRetry(url)) {
            await new Promise(resolve => setTimeout(resolve, delay));
            return this.add(url, fn, options);
          }
        }

        throw error;
      }
    }, { priority: options.priority });
  }

  private getQueue(domain: string): PQueue {
    if (!this.queues.has(domain)) {
      const config = this.rateLimiter.getDomainConfig(domain);

      this.queues.set(domain, new PQueue({
        concurrency: config.rateLimit.limits.perSecond || 1,
        interval: 1000,
        intervalCap: config.rateLimit.limits.perSecond || 1
      }));
    }

    return this.queues.get(domain)!;
  }

  private getBackoff(domain: string): ExponentialBackoff {
    if (!this.backoffManager.has(domain)) {
      const config = this.rateLimiter.getDomainConfig(domain);

      this.backoffManager.set(domain, new ExponentialBackoff(
        config.backoffStrategy || {
          type: 'exponential',
          initialDelay: 1000,
          maxDelay: 60000,
          multiplier: 2,
          jitter: true,
          maxRetries: 3
        }
      ));
    }

    return this.backoffManager.get(domain)!;
  }

  private isRateLimitError(error: any): boolean {
    return error.statusCode === 429 ||
           error.message?.includes('rate limit') ||
           error.message?.includes('too many requests');
  }

  getStats(): QueueStats {
    const stats: QueueStats = {
      domains: {},
      totalPending: 0,
      totalActive: 0
    };

    this.queues.forEach((queue, domain) => {
      stats.domains[domain] = {
        pending: queue.pending,
        active: queue.size - queue.pending,
        concurrency: queue.concurrency
      };

      stats.totalPending += queue.pending;
      stats.totalActive += queue.size - queue.pending;
    });

    return stats;
  }
}
```

## Testing & Validation

### Test Scenarios

```typescript
describe('Rate Limiting System', () => {
  it('should respect per-second limits', async () => {
    const limiter = new AdaptiveRateLimiter(redis);
    const results = [];

    for (let i = 0; i < 10; i++) {
      results.push(await limiter.checkLimit('https://example.com/page'));
    }

    expect(results.filter(r => r.allowed).length).toBeLessThanOrEqual(2);
  });

  it('should adapt rate based on success', async () => {
    // Test adaptive rate adjustment
  });

  it('should trigger circuit breaker on failures', async () => {
    // Test circuit breaker activation
  });

  it('should apply exponential backoff correctly', async () => {
    // Test backoff calculations
  });
});
```

## Monitoring & Metrics

```typescript
interface RateLimitMetrics {
  domains: Map<string, {
    requests: number;
    allowed: number;
    blocked: number;
    currentRate: number;
    successRate: number;
    circuitBreakerState: string;
  }>;
  totalRequests: number;
  totalBlocked: number;
  avgWaitTime: number;
}
```

## Configuration Examples

```json
{
  "domains": [
    {
      "domain": "api.example.com",
      "rateLimit": {
        "type": "adaptive",
        "limits": {
          "perSecond": 5,
          "perMinute": 100,
          "burst": 10
        },
        "adaptiveConfig": {
          "targetSuccessRate": 0.95,
          "adjustmentFactor": 0.2,
          "minRate": 1,
          "maxRate": 10,
          "windowSize": 60000
        }
      }
    }
  ]
}
```

## Implementation Timeline

- **Week 1**: Core rate limiter with token bucket and sliding window
- **Week 2**: Exponential backoff and circuit breaker
- **Week 3**: Domain-specific configs and adaptive throttling
- **Week 4**: Integration with request queue and monitoring

## Success Criteria

1. **Zero 429 errors** from proper rate limiting
2. **95% success rate** with adaptive throttling
3. **<100ms overhead** per request
4. **Automatic recovery** from rate limit blocks
5. **Domain-specific optimization** working correctly

## Dependencies

```json
{
  "p-queue": "^8.0.1",
  "ioredis": "^5.4.1",
  "rate-limiter-flexible": "^5.0.3"
}
```