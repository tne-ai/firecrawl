# Advanced Error Recovery & Resilience Implementation Plan

## Executive Summary
Implement a comprehensive error recovery system with intelligent retry logic, fallback mechanisms, error classification, and self-healing capabilities. This plan transforms basic error handling into a resilient system that automatically recovers from failures and learns from patterns.

## Current State Analysis
- **Location**: `apps/api/src/scraper/scrapeURL/error.ts`
- **Current Implementation**:
  - Basic error types defined
  - Engine waterfall fallback exists
  - Limited retry logic (3 attempts in fire-engine)
  - No intelligent error classification
  - No pattern learning

## Error Classification System

### Error Categories

```typescript
enum ErrorCategory {
  // Temporary errors - should retry
  TEMPORARY_NETWORK = 'temporary_network',
  TEMPORARY_SERVER = 'temporary_server',
  TEMPORARY_RESOURCE = 'temporary_resource',
  RATE_LIMIT = 'rate_limit',

  // Permanent errors - should not retry
  PERMANENT_CLIENT = 'permanent_client',
  PERMANENT_AUTH = 'permanent_auth',
  PERMANENT_NOT_FOUND = 'permanent_not_found',

  // Content errors - may need different approach
  CONTENT_BLOCKED = 'content_blocked',
  CONTENT_CAPTCHA = 'content_captcha',
  CONTENT_PAYWALL = 'content_paywall',

  // System errors - internal issues
  SYSTEM_RESOURCE = 'system_resource',
  SYSTEM_CONFIG = 'system_config'
}
```

## Implementation Architecture

### Phase 1: Intelligent Error Classification (Week 1)

#### 1.1 Error Classifier Service

**File**: `apps/api/src/services/error-recovery/ErrorClassifier.ts`

```typescript
export interface ErrorContext {
  error: Error;
  url: string;
  engine: string;
  statusCode?: number;
  headers?: Record<string, string>;
  responseBody?: string;
  attemptNumber: number;
  timestamp: number;
}

export interface ClassificationResult {
  category: ErrorCategory;
  isRetryable: boolean;
  suggestedAction: ErrorAction;
  confidence: number;
  metadata: Record<string, any>;
}

export enum ErrorAction {
  RETRY_IMMEDIATE = 'retry_immediate',
  RETRY_WITH_DELAY = 'retry_with_delay',
  RETRY_WITH_DIFFERENT_ENGINE = 'retry_different_engine',
  RETRY_WITH_PROXY = 'retry_with_proxy',
  RETRY_WITH_STEALTH = 'retry_with_stealth',
  SOLVE_CAPTCHA = 'solve_captcha',
  CHANGE_LOCATION = 'change_location',
  REDUCE_RATE = 'reduce_rate',
  ESCALATE = 'escalate',
  FAIL_PERMANENT = 'fail_permanent'
}

export class ErrorClassifier {
  private patterns: Map<string, ErrorPattern> = new Map();
  private mlModel?: ErrorClassificationModel;

  constructor() {
    this.loadErrorPatterns();
    this.loadMLModel();
  }

  async classify(context: ErrorContext): Promise<ClassificationResult> {
    // Try pattern-based classification first
    const patternResult = this.classifyByPattern(context);

    if (patternResult.confidence > 0.8) {
      return patternResult;
    }

    // Use ML model for complex cases
    if (this.mlModel) {
      const mlResult = await this.classifyWithML(context);
      if (mlResult.confidence > patternResult.confidence) {
        return mlResult;
      }
    }

    // Fallback to heuristics
    return this.classifyWithHeuristics(context);
  }

  private classifyByPattern(context: ErrorContext): ClassificationResult {
    const errorMessage = context.error.message.toLowerCase();
    const errorCode = (context.error as any).code;

    // Network errors
    if (errorCode === 'ECONNREFUSED' || errorCode === 'ENOTFOUND') {
      return {
        category: ErrorCategory.TEMPORARY_NETWORK,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_DELAY,
        confidence: 0.95,
        metadata: { errorCode }
      };
    }

    if (errorCode === 'ETIMEDOUT' || errorCode === 'ECONNRESET') {
      return {
        category: ErrorCategory.TEMPORARY_NETWORK,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_DIFFERENT_ENGINE,
        confidence: 0.9,
        metadata: { errorCode }
      };
    }

    // HTTP status codes
    if (context.statusCode) {
      return this.classifyByStatusCode(context);
    }

    // Content patterns
    if (context.responseBody) {
      return this.classifyByContent(context);
    }

    // Default classification
    return {
      category: ErrorCategory.TEMPORARY_SERVER,
      isRetryable: true,
      suggestedAction: ErrorAction.RETRY_WITH_DELAY,
      confidence: 0.3,
      metadata: {}
    };
  }

  private classifyByStatusCode(context: ErrorContext): ClassificationResult {
    const statusCode = context.statusCode!;

    const statusClassifications: Record<number, Partial<ClassificationResult>> = {
      // 4xx Client errors
      400: {
        category: ErrorCategory.PERMANENT_CLIENT,
        isRetryable: false,
        suggestedAction: ErrorAction.FAIL_PERMANENT
      },
      401: {
        category: ErrorCategory.PERMANENT_AUTH,
        isRetryable: false,
        suggestedAction: ErrorAction.FAIL_PERMANENT
      },
      403: {
        category: ErrorCategory.CONTENT_BLOCKED,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_PROXY
      },
      404: {
        category: ErrorCategory.PERMANENT_NOT_FOUND,
        isRetryable: false,
        suggestedAction: ErrorAction.FAIL_PERMANENT
      },
      429: {
        category: ErrorCategory.RATE_LIMIT,
        isRetryable: true,
        suggestedAction: ErrorAction.REDUCE_RATE
      },

      // 5xx Server errors
      500: {
        category: ErrorCategory.TEMPORARY_SERVER,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_DELAY
      },
      502: {
        category: ErrorCategory.TEMPORARY_SERVER,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_DIFFERENT_ENGINE
      },
      503: {
        category: ErrorCategory.TEMPORARY_RESOURCE,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_DELAY
      },
      504: {
        category: ErrorCategory.TEMPORARY_NETWORK,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_IMMEDIATE
      }
    };

    const classification = statusClassifications[statusCode];

    if (classification) {
      return {
        ...classification,
        confidence: 0.85,
        metadata: { statusCode }
      } as ClassificationResult;
    }

    // Generic classification by status range
    if (statusCode >= 400 && statusCode < 500) {
      return {
        category: ErrorCategory.PERMANENT_CLIENT,
        isRetryable: false,
        suggestedAction: ErrorAction.FAIL_PERMANENT,
        confidence: 0.6,
        metadata: { statusCode }
      };
    }

    if (statusCode >= 500) {
      return {
        category: ErrorCategory.TEMPORARY_SERVER,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_DELAY,
        confidence: 0.7,
        metadata: { statusCode }
      };
    }

    return {
      category: ErrorCategory.TEMPORARY_SERVER,
      isRetryable: true,
      suggestedAction: ErrorAction.RETRY_WITH_DELAY,
      confidence: 0.3,
      metadata: { statusCode }
    };
  }

  private classifyByContent(context: ErrorContext): ClassificationResult {
    const content = context.responseBody!.toLowerCase();

    // CAPTCHA detection
    if (content.includes('captcha') || content.includes('challenge')) {
      return {
        category: ErrorCategory.CONTENT_CAPTCHA,
        isRetryable: true,
        suggestedAction: ErrorAction.SOLVE_CAPTCHA,
        confidence: 0.9,
        metadata: { detectedType: 'captcha' }
      };
    }

    // Cloudflare detection
    if (content.includes('cloudflare') && content.includes('checking your browser')) {
      return {
        category: ErrorCategory.CONTENT_BLOCKED,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_STEALTH,
        confidence: 0.95,
        metadata: { detectedType: 'cloudflare' }
      };
    }

    // Rate limiting messages
    if (content.includes('rate limit') || content.includes('too many requests')) {
      return {
        category: ErrorCategory.RATE_LIMIT,
        isRetryable: true,
        suggestedAction: ErrorAction.REDUCE_RATE,
        confidence: 0.85,
        metadata: { detectedType: 'rate_limit' }
      };
    }

    // Access denied
    if (content.includes('access denied') || content.includes('forbidden')) {
      return {
        category: ErrorCategory.CONTENT_BLOCKED,
        isRetryable: true,
        suggestedAction: ErrorAction.RETRY_WITH_PROXY,
        confidence: 0.8,
        metadata: { detectedType: 'access_denied' }
      };
    }

    return {
      category: ErrorCategory.TEMPORARY_SERVER,
      isRetryable: true,
      suggestedAction: ErrorAction.RETRY_WITH_DELAY,
      confidence: 0.4,
      metadata: {}
    };
  }

  async recordOutcome(
    context: ErrorContext,
    classification: ClassificationResult,
    outcome: 'success' | 'failure'
  ): Promise<void> {
    // Learn from outcomes to improve classification
    await this.updatePatterns(context, classification, outcome);

    if (this.mlModel) {
      await this.mlModel.train(context, classification, outcome);
    }
  }
}
```

### Phase 2: Retry Strategy Manager (Week 1-2)

#### 2.1 Advanced Retry Logic

**File**: `apps/api/src/services/error-recovery/RetryManager.ts`

```typescript
export interface RetryStrategy {
  maxAttempts: number;
  baseDelay: number;
  maxDelay: number;
  backoffMultiplier: number;
  jitter: boolean;
  retryCondition: (error: Error, attempt: number) => boolean;
  onRetry?: (error: Error, attempt: number) => void;
}

export class RetryManager {
  private strategies: Map<ErrorCategory, RetryStrategy> = new Map();
  private attemptHistory: Map<string, AttemptHistory> = new Map();

  constructor() {
    this.initializeStrategies();
  }

  private initializeStrategies(): void {
    // Temporary network errors - aggressive retry
    this.strategies.set(ErrorCategory.TEMPORARY_NETWORK, {
      maxAttempts: 5,
      baseDelay: 500,
      maxDelay: 10000,
      backoffMultiplier: 2,
      jitter: true,
      retryCondition: (error, attempt) => attempt <= 5
    });

    // Rate limiting - exponential backoff
    this.strategies.set(ErrorCategory.RATE_LIMIT, {
      maxAttempts: 10,
      baseDelay: 2000,
      maxDelay: 120000,
      backoffMultiplier: 3,
      jitter: false,
      retryCondition: (error, attempt) => attempt <= 10,
      onRetry: (error, attempt) => {
        logger.info('Rate limit retry', { attempt, nextDelay: this.calculateDelay(attempt, this.strategies.get(ErrorCategory.RATE_LIMIT)!) });
      }
    });

    // Server errors - moderate retry
    this.strategies.set(ErrorCategory.TEMPORARY_SERVER, {
      maxAttempts: 3,
      baseDelay: 1000,
      maxDelay: 30000,
      backoffMultiplier: 2.5,
      jitter: true,
      retryCondition: (error, attempt) => attempt <= 3
    });

    // CAPTCHA - single retry after solving
    this.strategies.set(ErrorCategory.CONTENT_CAPTCHA, {
      maxAttempts: 2,
      baseDelay: 0,
      maxDelay: 0,
      backoffMultiplier: 1,
      jitter: false,
      retryCondition: (error, attempt) => attempt === 1
    });
  }

  async executeWithRetry<T>(
    operation: () => Promise<T>,
    context: {
      url: string;
      errorClassifier: ErrorClassifier;
      meta: Meta;
    }
  ): Promise<T> {
    const operationKey = `${context.url}:${Date.now()}`;
    let lastError: Error | null = null;
    let attempt = 0;

    while (attempt < 10) { // Hard limit
      attempt++;

      try {
        const result = await operation();

        // Success - record and return
        this.recordSuccess(operationKey);
        return result;
      } catch (error) {
        lastError = error as Error;

        // Classify the error
        const classification = await context.errorClassifier.classify({
          error: lastError,
          url: context.url,
          engine: context.meta.winnerEngine || 'unknown',
          attemptNumber: attempt,
          timestamp: Date.now()
        });

        // Check if we should retry
        if (!classification.isRetryable) {
          throw error;
        }

        const strategy = this.strategies.get(classification.category);
        if (!strategy || !strategy.retryCondition(lastError, attempt)) {
          throw error;
        }

        // Apply suggested action
        await this.applySuggestedAction(
          classification.suggestedAction,
          context
        );

        // Calculate and apply delay
        const delay = this.calculateDelay(attempt, strategy);
        await new Promise(resolve => setTimeout(resolve, delay));

        // Call retry callback if exists
        if (strategy.onRetry) {
          strategy.onRetry(lastError, attempt);
        }

        // Record attempt
        this.recordAttempt(operationKey, classification);
      }
    }

    throw lastError || new Error('Max retry attempts exceeded');
  }

  private calculateDelay(attempt: number, strategy: RetryStrategy): number {
    let delay = strategy.baseDelay * Math.pow(strategy.backoffMultiplier, attempt - 1);

    // Apply max delay cap
    delay = Math.min(delay, strategy.maxDelay);

    // Apply jitter if enabled
    if (strategy.jitter) {
      const jitterRange = delay * 0.3;
      delay = delay + (Math.random() - 0.5) * jitterRange;
    }

    return Math.max(0, delay);
  }

  private async applySuggestedAction(
    action: ErrorAction,
    context: any
  ): Promise<void> {
    switch (action) {
      case ErrorAction.RETRY_WITH_PROXY:
        // Switch to a different proxy
        context.meta.options.proxy = 'rotate';
        break;

      case ErrorAction.RETRY_WITH_STEALTH:
        // Enable stealth mode
        context.meta.featureFlags.add('stealth');
        break;

      case ErrorAction.RETRY_WITH_DIFFERENT_ENGINE:
        // Force engine rotation
        context.meta.forceEngineRotation = true;
        break;

      case ErrorAction.SOLVE_CAPTCHA:
        // Trigger CAPTCHA solving
        context.meta.featureFlags.add('solveCaptcha');
        break;

      case ErrorAction.REDUCE_RATE:
        // Reduce request rate
        await this.reduceRate(context.url);
        break;

      case ErrorAction.CHANGE_LOCATION:
        // Change geo location
        context.meta.options.location = this.selectNewLocation();
        break;
    }
  }

  private recordAttempt(key: string, classification: ClassificationResult): void {
    if (!this.attemptHistory.has(key)) {
      this.attemptHistory.set(key, {
        attempts: [],
        startTime: Date.now()
      });
    }

    const history = this.attemptHistory.get(key)!;
    history.attempts.push({
      timestamp: Date.now(),
      category: classification.category,
      action: classification.suggestedAction
    });
  }

  private recordSuccess(key: string): void {
    const history = this.attemptHistory.get(key);
    if (history) {
      const duration = Date.now() - history.startTime;
      logger.info('Operation succeeded after retries', {
        attempts: history.attempts.length,
        duration,
        patterns: history.attempts.map(a => a.category)
      });
    }
  }
}
```

### Phase 3: Self-Healing System (Week 2)

#### 3.1 Pattern Learning & Adaptation

**File**: `apps/api/src/services/error-recovery/SelfHealingSystem.ts`

```typescript
export class SelfHealingSystem {
  private errorPatterns: Map<string, ErrorPattern> = new Map();
  private healingStrategies: Map<string, HealingStrategy> = new Map();
  private redis: Redis;

  constructor(redis: Redis) {
    this.redis = redis;
    this.loadPatterns();
  }

  async analyzeAndHeal(
    domain: string,
    errors: ErrorContext[]
  ): Promise<HealingAction[]> {
    // Analyze error patterns
    const patterns = this.identifyPatterns(errors);

    // Generate healing actions
    const actions: HealingAction[] = [];

    for (const pattern of patterns) {
      const strategy = await this.selectHealingStrategy(domain, pattern);
      if (strategy) {
        actions.push(...strategy.generateActions(pattern));
      }
    }

    // Apply healing actions
    for (const action of actions) {
      await this.applyHealingAction(domain, action);
    }

    return actions;
  }

  private identifyPatterns(errors: ErrorContext[]): ErrorPattern[] {
    const patterns: ErrorPattern[] = [];

    // Time-based patterns
    const timePattern = this.analyzeTimePattern(errors);
    if (timePattern) patterns.push(timePattern);

    // Frequency patterns
    const frequencyPattern = this.analyzeFrequencyPattern(errors);
    if (frequencyPattern) patterns.push(frequencyPattern);

    // Sequence patterns
    const sequencePattern = this.analyzeSequencePattern(errors);
    if (sequencePattern) patterns.push(sequencePattern);

    // Correlation patterns
    const correlationPattern = this.analyzeCorrelationPattern(errors);
    if (correlationPattern) patterns.push(correlationPattern);

    return patterns;
  }

  private analyzeTimePattern(errors: ErrorContext[]): ErrorPattern | null {
    // Group errors by hour of day
    const hourlyErrors = new Map<number, number>();

    errors.forEach(error => {
      const hour = new Date(error.timestamp).getHours();
      hourlyErrors.set(hour, (hourlyErrors.get(hour) || 0) + 1);
    });

    // Find peak error hours
    const peakHours = Array.from(hourlyErrors.entries())
      .filter(([_, count]) => count > errors.length * 0.2)
      .map(([hour]) => hour);

    if (peakHours.length > 0) {
      return {
        type: 'time-based',
        confidence: 0.8,
        description: `High error rate during hours: ${peakHours.join(', ')}`,
        data: { peakHours }
      };
    }

    return null;
  }

  private async selectHealingStrategy(
    domain: string,
    pattern: ErrorPattern
  ): Promise<HealingStrategy | null> {
    // Get historical success rates for different strategies
    const history = await this.getStrategyHistory(domain, pattern.type);

    // Select best performing strategy
    let bestStrategy: HealingStrategy | null = null;
    let bestSuccessRate = 0;

    for (const [strategyName, strategy] of this.healingStrategies) {
      const successRate = history[strategyName]?.successRate || 0.5;

      if (successRate > bestSuccessRate && strategy.canHandle(pattern)) {
        bestStrategy = strategy;
        bestSuccessRate = successRate;
      }
    }

    return bestStrategy;
  }

  private async applyHealingAction(
    domain: string,
    action: HealingAction
  ): Promise<void> {
    logger.info('Applying healing action', {
      domain,
      action: action.type,
      parameters: action.parameters
    });

    switch (action.type) {
      case 'adjust-rate-limit':
        await this.adjustRateLimit(domain, action.parameters);
        break;

      case 'enable-proxy':
        await this.enableProxy(domain, action.parameters);
        break;

      case 'change-engine':
        await this.changeDefaultEngine(domain, action.parameters);
        break;

      case 'add-delay':
        await this.addCrawlDelay(domain, action.parameters);
        break;

      case 'enable-stealth':
        await this.enableStealthMode(domain, action.parameters);
        break;

      case 'blacklist-time':
        await this.blacklistTimeWindow(domain, action.parameters);
        break;
    }

    // Record action for learning
    await this.recordHealingAction(domain, action);
  }

  async learnFromOutcome(
    domain: string,
    action: HealingAction,
    outcome: 'success' | 'failure' | 'partial'
  ): Promise<void> {
    const key = `healing:${domain}:${action.type}`;
    const stats = await this.redis.hgetall(key);

    const total = parseInt(stats.total || '0') + 1;
    const successes = parseInt(stats.successes || '0') + (outcome === 'success' ? 1 : 0);
    const successRate = successes / total;

    await this.redis.hmset(key, {
      total: total.toString(),
      successes: successes.toString(),
      successRate: successRate.toString(),
      lastUsed: Date.now().toString()
    });

    // Adjust strategy confidence based on outcome
    const strategy = this.healingStrategies.get(action.type);
    if (strategy) {
      strategy.adjustConfidence(outcome);
    }
  }
}

interface ErrorPattern {
  type: string;
  confidence: number;
  description: string;
  data: any;
}

interface HealingAction {
  type: string;
  parameters: any;
  priority: number;
  estimatedImpact: number;
}

interface HealingStrategy {
  canHandle(pattern: ErrorPattern): boolean;
  generateActions(pattern: ErrorPattern): HealingAction[];
  adjustConfidence(outcome: string): void;
}

class RateLimitHealingStrategy implements HealingStrategy {
  private confidence = 0.7;

  canHandle(pattern: ErrorPattern): boolean {
    return pattern.type === 'frequency' &&
           pattern.data.errorRate > 0.3;
  }

  generateActions(pattern: ErrorPattern): HealingAction[] {
    const currentRate = pattern.data.currentRate || 1;
    const targetRate = currentRate * 0.5; // Reduce by 50%

    return [{
      type: 'adjust-rate-limit',
      parameters: {
        newRate: targetRate,
        duration: 3600000 // 1 hour
      },
      priority: 1,
      estimatedImpact: 0.8
    }];
  }

  adjustConfidence(outcome: string): void {
    if (outcome === 'success') {
      this.confidence = Math.min(1, this.confidence * 1.1);
    } else if (outcome === 'failure') {
      this.confidence = Math.max(0.1, this.confidence * 0.9);
    }
  }
}
```

### Phase 4: Fallback Chain Manager (Week 2)

#### 4.1 Engine Fallback System

**File**: `apps/api/src/services/error-recovery/FallbackManager.ts`

```typescript
export class FallbackManager {
  private engineChains: Map<string, EngineChain> = new Map();
  private engineStats: Map<string, EngineStats> = new Map();

  async executeWithFallback<T>(
    operation: EngineOperation<T>,
    meta: Meta
  ): Promise<T> {
    const chain = this.getEngineChain(meta.url);
    const engines = chain.getOrderedEngines();

    let lastError: Error | null = null;

    for (const engine of engines) {
      try {
        // Check if engine is healthy
        if (!this.isEngineHealthy(engine, meta.url)) {
          continue;
        }

        // Execute with engine
        const result = await operation.execute(engine, meta);

        // Record success
        this.recordEngineSuccess(engine, meta.url);

        return result;
      } catch (error) {
        lastError = error as Error;

        // Record failure
        this.recordEngineFailure(engine, meta.url, error);

        // Check if we should continue
        if (!this.shouldContinueFallback(error)) {
          throw error;
        }

        // Log fallback
        logger.info('Engine failed, trying next', {
          failedEngine: engine,
          nextEngine: engines[engines.indexOf(engine) + 1],
          error: error.message
        });
      }
    }

    throw lastError || new Error('All engines failed');
  }

  private getEngineChain(url: string): EngineChain {
    const domain = new URL(url).hostname;

    if (!this.engineChains.has(domain)) {
      this.engineChains.set(domain, this.buildEngineChain(domain));
    }

    return this.engineChains.get(domain)!;
  }

  private buildEngineChain(domain: string): EngineChain {
    const stats = this.getEngineStatsForDomain(domain);

    // Sort engines by success rate for this domain
    const engines = ['tls-fetch', 'stealth-browser', 'playwright', 'fetch']
      .sort((a, b) => {
        const aStats = stats[a] || { successRate: 0.5 };
        const bStats = stats[b] || { successRate: 0.5 };
        return bStats.successRate - aStats.successRate;
      });

    return new EngineChain(engines);
  }

  private isEngineHealthy(engine: string, url: string): boolean {
    const stats = this.engineStats.get(`${engine}:${new URL(url).hostname}`);

    if (!stats) return true; // No stats, assume healthy

    // Check recent failure rate
    const recentFailureRate = stats.recentFailures / (stats.recentAttempts || 1);
    if (recentFailureRate > 0.8) {
      return false;
    }

    // Check if in cooldown
    if (stats.cooldownUntil && stats.cooldownUntil > Date.now()) {
      return false;
    }

    return true;
  }

  private shouldContinueFallback(error: any): boolean {
    // Don't fallback for permanent errors
    if (error.statusCode === 404 || error.statusCode === 401) {
      return false;
    }

    // Don't fallback for CAPTCHA (needs solving, not engine change)
    if (error.type === 'CAPTCHA_REQUIRED') {
      return false;
    }

    return true;
  }

  private recordEngineSuccess(engine: string, url: string): void {
    const key = `${engine}:${new URL(url).hostname}`;

    if (!this.engineStats.has(key)) {
      this.engineStats.set(key, {
        totalAttempts: 0,
        totalSuccesses: 0,
        recentAttempts: 0,
        recentSuccesses: 0,
        recentFailures: 0,
        successRate: 1,
        cooldownUntil: null
      });
    }

    const stats = this.engineStats.get(key)!;
    stats.totalAttempts++;
    stats.totalSuccesses++;
    stats.recentAttempts++;
    stats.recentSuccesses++;
    stats.successRate = stats.totalSuccesses / stats.totalAttempts;

    // Reset recent window if needed
    if (stats.recentAttempts > 10) {
      stats.recentAttempts = 0;
      stats.recentSuccesses = 0;
      stats.recentFailures = 0;
    }
  }

  private recordEngineFailure(engine: string, url: string, error: any): void {
    const key = `${engine}:${new URL(url).hostname}`;

    if (!this.engineStats.has(key)) {
      this.engineStats.set(key, {
        totalAttempts: 0,
        totalSuccesses: 0,
        recentAttempts: 0,
        recentSuccesses: 0,
        recentFailures: 0,
        successRate: 0,
        cooldownUntil: null
      });
    }

    const stats = this.engineStats.get(key)!;
    stats.totalAttempts++;
    stats.recentAttempts++;
    stats.recentFailures++;
    stats.successRate = stats.totalSuccesses / stats.totalAttempts;

    // Apply cooldown if too many recent failures
    if (stats.recentFailures >= 5) {
      stats.cooldownUntil = Date.now() + 60000; // 1 minute cooldown
    }
  }
}
```

## Testing & Validation

### Test Scenarios

```typescript
describe('Error Recovery System', () => {
  it('should classify errors correctly', async () => {
    const classifier = new ErrorClassifier();

    const result = await classifier.classify({
      error: new Error('ECONNREFUSED'),
      url: 'https://example.com',
      engine: 'fetch',
      attemptNumber: 1,
      timestamp: Date.now()
    });

    expect(result.category).toBe(ErrorCategory.TEMPORARY_NETWORK);
    expect(result.isRetryable).toBe(true);
  });

  it('should apply exponential backoff', async () => {
    // Test retry delays increase exponentially
  });

  it('should learn from patterns', async () => {
    // Test self-healing system learns
  });

  it('should fallback through engines', async () => {
    // Test engine fallback chain
  });
});
```

## Monitoring & Metrics

```typescript
interface ErrorRecoveryMetrics {
  totalErrors: number;
  recoveredErrors: number;
  permanentFailures: number;
  recoveryRate: number;
  avgRecoveryTime: number;
  errorsByCategory: Map<ErrorCategory, number>;
  healingActionsApplied: number;
  engineFailoverCount: number;
}
```

## Implementation Timeline

- **Week 1**: Error classification and retry strategies
- **Week 2**: Self-healing system and fallback management
- **Week 3**: Pattern learning and adaptation
- **Week 4**: Integration and monitoring

## Success Criteria

1. **90% error recovery rate** for temporary errors
2. **<5 seconds average recovery time**
3. **Zero infinite retry loops**
4. **Accurate error classification** (>95% accuracy)
5. **Self-healing effectiveness** (>70% success rate)

## Dependencies

```json
{
  "p-retry": "^6.2.0",
  "bottleneck": "^2.19.5",
  "async-retry": "^1.3.3"
}
```