# Browser Pool Management Implementation Plan

## Executive Summary
Implement an advanced browser pool management system using puppeteer-cluster/playwright-cluster patterns, with intelligent resource management, session persistence, and automatic scaling. This addresses the critical need for efficient browser instance management at scale.

## Current State Analysis
- **Current Implementation**: Basic Playwright service with single browser instance
- **Limitations**:
  - No browser pooling
  - No session reuse
  - High memory consumption (300MB-1GB per instance)
  - No automatic cleanup of zombie processes
  - No load balancing across instances

## Key Technologies

### Recommended Solutions

1. **Puppeteer-Cluster**: Proven Node.js library for browser pool management
2. **Playwright Browser Pool**: Custom implementation with context isolation
3. **Generic Pool**: Low-level pooling library for custom solutions
4. **Kubernetes Job Queue**: For distributed browser automation

## Implementation Architecture

### Phase 1: Core Browser Pool System (Week 1)

#### 1.1 Browser Pool Manager

**File**: `apps/api/src/services/browser-pool/BrowserPoolManager.ts`

```typescript
import { Browser, BrowserContext, Page, chromium, firefox, webkit } from 'playwright';
import { EventEmitter } from 'events';

export interface PoolConfig {
  minInstances: number;
  maxInstances: number;
  maxContextsPerBrowser: number;
  maxPagesPerContext: number;
  browserType: 'chromium' | 'firefox' | 'webkit';
  launchOptions: any;
  contextOptions: any;
  recycleThreshold: number;  // Pages before recycling browser
  idleTimeoutMs: number;
  healthCheckIntervalMs: number;
  resourceLimits: {
    maxMemoryMB: number;
    maxCPUPercent: number;
  };
}

export interface BrowserInstance {
  id: string;
  browser: Browser;
  contexts: Map<string, ContextInstance>;
  pageCount: number;
  createdAt: Date;
  lastUsedAt: Date;
  totalPagesServed: number;
  memoryUsage: number;
  cpuUsage: number;
  isHealthy: boolean;
}

export interface ContextInstance {
  id: string;
  context: BrowserContext;
  pages: Map<string, Page>;
  createdAt: Date;
  lastUsedAt: Date;
}

export class BrowserPoolManager extends EventEmitter {
  private config: PoolConfig;
  private browsers: Map<string, BrowserInstance> = new Map();
  private waitingQueue: Array<(page: Page) => void> = [];
  private metrics: PoolMetrics;
  private healthChecker: NodeJS.Timer;
  private scaler: AutoScaler;

  constructor(config: PoolConfig) {
    super();
    this.config = config;
    this.metrics = new PoolMetrics();
    this.scaler = new AutoScaler(this);
    this.initialize();
  }

  private async initialize(): Promise<void> {
    // Create minimum browser instances
    for (let i = 0; i < this.config.minInstances; i++) {
      await this.createBrowserInstance();
    }

    // Start health checking
    this.startHealthCheck();

    // Start auto-scaling
    this.scaler.start();

    logger.info('Browser pool initialized', {
      minInstances: this.config.minInstances,
      maxInstances: this.config.maxInstances
    });
  }

  async acquire(options?: {
    contextOptions?: any;
    pageOptions?: any;
    sessionId?: string;
  }): Promise<Page> {
    const startTime = Date.now();

    // Try to find available page
    const page = await this.findAvailablePage(options?.sessionId);

    if (page) {
      this.metrics.recordAcquisition(Date.now() - startTime);
      return page;
    }

    // No available page, check if we can create more
    if (this.canCreateMore()) {
      const newPage = await this.createNewPage(options);
      this.metrics.recordAcquisition(Date.now() - startTime);
      return newPage;
    }

    // Wait for available page
    return new Promise((resolve) => {
      this.waitingQueue.push((page) => {
        this.metrics.recordAcquisition(Date.now() - startTime);
        resolve(page);
      });

      // Timeout after 30 seconds
      setTimeout(() => {
        const index = this.waitingQueue.indexOf(resolve as any);
        if (index > -1) {
          this.waitingQueue.splice(index, 1);
          throw new Error('Browser pool acquisition timeout');
        }
      }, 30000);
    });
  }

  async release(page: Page): Promise<void> {
    try {
      // Clear page state
      await this.clearPageState(page);

      // Check if browser needs recycling
      const browser = this.findBrowserByPage(page);
      if (browser && this.shouldRecycle(browser)) {
        await this.recycleBrowser(browser.id);
      } else {
        // Mark page as available
        this.markPageAvailable(page);

        // Process waiting queue
        if (this.waitingQueue.length > 0) {
          const callback = this.waitingQueue.shift();
          if (callback) {
            callback(page);
          }
        }
      }

      this.metrics.recordRelease();
    } catch (error) {
      logger.error('Failed to release page', { error });
      await this.handleCorruptedPage(page);
    }
  }

  private async createBrowserInstance(): Promise<BrowserInstance> {
    const id = this.generateId();
    const browserType = this.getBrowserType();

    const browser = await browserType.launch({
      ...this.config.launchOptions,
      handleSIGINT: false,
      handleSIGTERM: false,
      handleSIGHUP: false
    });

    const instance: BrowserInstance = {
      id,
      browser,
      contexts: new Map(),
      pageCount: 0,
      createdAt: new Date(),
      lastUsedAt: new Date(),
      totalPagesServed: 0,
      memoryUsage: 0,
      cpuUsage: 0,
      isHealthy: true
    };

    this.browsers.set(id, instance);

    this.emit('browser-created', { id });

    return instance;
  }

  private async createNewPage(options?: any): Promise<Page> {
    // Find browser with capacity
    let browser = this.findBrowserWithCapacity();

    if (!browser) {
      // Create new browser if under limit
      if (this.browsers.size < this.config.maxInstances) {
        browser = await this.createBrowserInstance();
      } else {
        throw new Error('No browser capacity available');
      }
    }

    // Find or create context
    let context = this.findContextWithCapacity(browser);

    if (!context) {
      context = await this.createContext(browser, options?.contextOptions);
    }

    // Create page
    const page = await context.context.newPage(options?.pageOptions || {});

    // Track page
    context.pages.set(this.generateId(), page);
    browser.pageCount++;
    browser.totalPagesServed++;
    browser.lastUsedAt = new Date();

    return page;
  }

  private async createContext(
    browser: BrowserInstance,
    options?: any
  ): Promise<ContextInstance> {
    const id = this.generateId();

    const context = await browser.browser.newContext({
      ...this.config.contextOptions,
      ...options
    });

    const instance: ContextInstance = {
      id,
      context,
      pages: new Map(),
      createdAt: new Date(),
      lastUsedAt: new Date()
    };

    browser.contexts.set(id, instance);

    return instance;
  }

  private findAvailablePage(sessionId?: string): Page | null {
    // If session ID provided, try to find existing context
    if (sessionId) {
      for (const browser of this.browsers.values()) {
        for (const context of browser.contexts.values()) {
          if (context.id === sessionId && context.pages.size < this.config.maxPagesPerContext) {
            // Reuse context for session
            return null; // Will create new page in same context
          }
        }
      }
    }

    // Find any available page
    for (const browser of this.browsers.values()) {
      if (!browser.isHealthy) continue;

      for (const context of browser.contexts.values()) {
        for (const [pageId, page] of context.pages) {
          if (!this.isPageBusy(page)) {
            return page;
          }
        }
      }
    }

    return null;
  }

  private findBrowserWithCapacity(): BrowserInstance | null {
    for (const browser of this.browsers.values()) {
      if (!browser.isHealthy) continue;

      if (browser.contexts.size < this.config.maxContextsPerBrowser) {
        return browser;
      }

      // Check if any context has capacity
      for (const context of browser.contexts.values()) {
        if (context.pages.size < this.config.maxPagesPerContext) {
          return browser;
        }
      }
    }

    return null;
  }

  private findContextWithCapacity(browser: BrowserInstance): ContextInstance | null {
    for (const context of browser.contexts.values()) {
      if (context.pages.size < this.config.maxPagesPerContext) {
        return context;
      }
    }
    return null;
  }

  private shouldRecycle(browser: BrowserInstance): boolean {
    // Recycle if too many pages served
    if (browser.totalPagesServed > this.config.recycleThreshold) {
      return true;
    }

    // Recycle if memory usage too high
    if (browser.memoryUsage > this.config.resourceLimits.maxMemoryMB) {
      return true;
    }

    // Recycle if browser is old
    const age = Date.now() - browser.createdAt.getTime();
    if (age > 3600000) { // 1 hour
      return true;
    }

    return false;
  }

  private async recycleBrowser(browserId: string): Promise<void> {
    const browser = this.browsers.get(browserId);
    if (!browser) return;

    logger.info('Recycling browser', {
      id: browserId,
      pagesServed: browser.totalPagesServed,
      memoryUsage: browser.memoryUsage
    });

    try {
      // Close all contexts and pages
      for (const context of browser.contexts.values()) {
        await context.context.close();
      }

      // Close browser
      await browser.browser.close();

      // Remove from pool
      this.browsers.delete(browserId);

      // Create replacement if needed
      if (this.browsers.size < this.config.minInstances) {
        await this.createBrowserInstance();
      }

      this.emit('browser-recycled', { id: browserId });
    } catch (error) {
      logger.error('Failed to recycle browser', { error, browserId });
      this.handleZombieBrowser(browserId);
    }
  }

  private async clearPageState(page: Page): Promise<void> {
    try {
      // Navigate to blank page
      await page.goto('about:blank', { waitUntil: 'domcontentloaded' });

      // Clear cookies
      await page.context().clearCookies();

      // Clear local storage
      await page.evaluate(() => {
        try {
          localStorage.clear();
          sessionStorage.clear();
        } catch (e) {
          // Ignore errors
        }
      });

      // Clear cache if possible
      if (page.context().clearCache) {
        await page.context().clearCache();
      }
    } catch (error) {
      logger.warn('Failed to clear page state', { error });
    }
  }

  private startHealthCheck(): void {
    this.healthChecker = setInterval(async () => {
      for (const browser of this.browsers.values()) {
        await this.checkBrowserHealth(browser);
      }
    }, this.config.healthCheckIntervalMs);
  }

  private async checkBrowserHealth(browser: BrowserInstance): Promise<void> {
    try {
      // Check if browser is responsive
      const contexts = browser.browser.contexts();

      // Check memory usage
      const metrics = await this.getBrowserMetrics(browser);
      browser.memoryUsage = metrics.memory;
      browser.cpuUsage = metrics.cpu;

      // Check if browser is zombie
      if (contexts.length === 0 && browser.contexts.size > 0) {
        browser.isHealthy = false;
        await this.handleZombieBrowser(browser.id);
        return;
      }

      // Check resource limits
      if (browser.memoryUsage > this.config.resourceLimits.maxMemoryMB) {
        logger.warn('Browser exceeding memory limit', {
          id: browser.id,
          memory: browser.memoryUsage
        });
        browser.isHealthy = false;
      }

      browser.isHealthy = true;
    } catch (error) {
      logger.error('Browser health check failed', {
        browserId: browser.id,
        error
      });
      browser.isHealthy = false;
    }
  }

  private async getBrowserMetrics(browser: BrowserInstance): Promise<{
    memory: number;
    cpu: number;
  }> {
    // Get browser process metrics
    // This would integrate with system monitoring tools
    return {
      memory: process.memoryUsage().heapUsed / 1024 / 1024,
      cpu: process.cpuUsage().system / 1000000
    };
  }

  private async handleZombieBrowser(browserId: string): Promise<void> {
    logger.warn('Handling zombie browser', { browserId });

    try {
      const browser = this.browsers.get(browserId);
      if (browser) {
        // Force kill browser process
        await browser.browser.close().catch(() => {});

        // Remove from pool
        this.browsers.delete(browserId);

        // Create replacement
        if (this.browsers.size < this.config.minInstances) {
          await this.createBrowserInstance();
        }
      }
    } catch (error) {
      logger.error('Failed to handle zombie browser', { error });
    }
  }

  private canCreateMore(): boolean {
    return this.browsers.size < this.config.maxInstances;
  }

  private getBrowserType() {
    switch (this.config.browserType) {
      case 'firefox':
        return firefox;
      case 'webkit':
        return webkit;
      default:
        return chromium;
    }
  }

  private generateId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  async shutdown(): Promise<void> {
    logger.info('Shutting down browser pool');

    // Clear health checker
    if (this.healthChecker) {
      clearInterval(this.healthChecker);
    }

    // Stop auto-scaler
    this.scaler.stop();

    // Close all browsers
    const closePromises = Array.from(this.browsers.values()).map(async (browser) => {
      try {
        await browser.browser.close();
      } catch (error) {
        logger.error('Failed to close browser', { error, id: browser.id });
      }
    });

    await Promise.all(closePromises);

    this.browsers.clear();
    this.waitingQueue = [];

    logger.info('Browser pool shutdown complete');
  }

  getStats(): PoolStats {
    return {
      totalBrowsers: this.browsers.size,
      healthyBrowsers: Array.from(this.browsers.values()).filter(b => b.isHealthy).length,
      totalContexts: Array.from(this.browsers.values()).reduce((sum, b) => sum + b.contexts.size, 0),
      totalPages: Array.from(this.browsers.values()).reduce((sum, b) => sum + b.pageCount, 0),
      waitingQueue: this.waitingQueue.length,
      metrics: this.metrics.getSnapshot()
    };
  }
}
```

### Phase 2: Auto-Scaling & Resource Management (Week 1-2)

#### 2.1 Auto-Scaler Implementation

**File**: `apps/api/src/services/browser-pool/AutoScaler.ts`

```typescript
export class AutoScaler {
  private pool: BrowserPoolManager;
  private scalingInterval: NodeJS.Timer;
  private metrics: ScalingMetrics;

  constructor(pool: BrowserPoolManager) {
    this.pool = pool;
    this.metrics = new ScalingMetrics();
  }

  start(): void {
    this.scalingInterval = setInterval(() => {
      this.evaluateScaling();
    }, 10000); // Check every 10 seconds
  }

  stop(): void {
    if (this.scalingInterval) {
      clearInterval(this.scalingInterval);
    }
  }

  private async evaluateScaling(): Promise<void> {
    const stats = this.pool.getStats();
    const decision = this.makeScalingDecision(stats);

    switch (decision) {
      case 'scale-up':
        await this.scaleUp();
        break;
      case 'scale-down':
        await this.scaleDown();
        break;
      case 'maintain':
        // No action needed
        break;
    }

    this.metrics.recordDecision(decision);
  }

  private makeScalingDecision(stats: PoolStats): ScalingDecision {
    const utilizationRate = stats.totalPages / (stats.totalBrowsers * this.pool.config.maxPagesPerContext * this.pool.config.maxContextsPerBrowser);
    const queuePressure = stats.waitingQueue / stats.totalBrowsers;

    // Scale up conditions
    if (utilizationRate > 0.8 || queuePressure > 2) {
      if (stats.totalBrowsers < this.pool.config.maxInstances) {
        return 'scale-up';
      }
    }

    // Scale down conditions
    if (utilizationRate < 0.3 && stats.totalBrowsers > this.pool.config.minInstances) {
      return 'scale-down';
    }

    return 'maintain';
  }

  private async scaleUp(): Promise<void> {
    logger.info('Scaling up browser pool');

    try {
      // Add new browser instances
      const instancesToAdd = Math.min(
        2, // Add max 2 at a time
        this.pool.config.maxInstances - this.pool.getStats().totalBrowsers
      );

      for (let i = 0; i < instancesToAdd; i++) {
        await this.pool.createBrowserInstance();
      }

      this.metrics.recordScaleUp(instancesToAdd);
    } catch (error) {
      logger.error('Failed to scale up', { error });
    }
  }

  private async scaleDown(): Promise<void> {
    logger.info('Scaling down browser pool');

    try {
      // Remove idle browsers
      const stats = this.pool.getStats();
      const instancesToRemove = Math.min(
        1, // Remove max 1 at a time
        stats.totalBrowsers - this.pool.config.minInstances
      );

      // Find and remove least recently used browsers
      const browsers = Array.from(this.pool.browsers.values())
        .sort((a, b) => a.lastUsedAt.getTime() - b.lastUsedAt.getTime());

      for (let i = 0; i < instancesToRemove; i++) {
        if (browsers[i]) {
          await this.pool.recycleBrowser(browsers[i].id);
        }
      }

      this.metrics.recordScaleDown(instancesToRemove);
    } catch (error) {
      logger.error('Failed to scale down', { error });
    }
  }
}
```

### Phase 3: Session Management (Week 2)

#### 3.1 Session Persistence

**File**: `apps/api/src/services/browser-pool/SessionManager.ts`

```typescript
export class SessionManager {
  private sessions: Map<string, BrowserSession> = new Map();
  private pool: BrowserPoolManager;

  constructor(pool: BrowserPoolManager) {
    this.pool = pool;
  }

  async createSession(options: {
    sessionId: string;
    cookies?: any[];
    localStorage?: Record<string, string>;
    auth?: { username: string; password: string };
  }): Promise<Page> {
    // Get page with session context
    const page = await this.pool.acquire({
      sessionId: options.sessionId,
      contextOptions: {
        httpCredentials: options.auth
      }
    });

    // Restore session state
    if (options.cookies) {
      await page.context().addCookies(options.cookies);
    }

    if (options.localStorage) {
      await page.evaluateOnNewDocument((storage) => {
        for (const [key, value] of Object.entries(storage)) {
          localStorage.setItem(key, value);
        }
      }, options.localStorage);
    }

    // Track session
    this.sessions.set(options.sessionId, {
      id: options.sessionId,
      page,
      createdAt: new Date(),
      lastAccessedAt: new Date()
    });

    return page;
  }

  async getSession(sessionId: string): Promise<Page | null> {
    const session = this.sessions.get(sessionId);

    if (session) {
      session.lastAccessedAt = new Date();
      return session.page;
    }

    return null;
  }

  async saveSessionState(sessionId: string): Promise<SessionState> {
    const session = this.sessions.get(sessionId);

    if (!session) {
      throw new Error('Session not found');
    }

    const cookies = await session.page.context().cookies();

    const localStorage = await session.page.evaluate(() => {
      const items: Record<string, string> = {};
      for (let i = 0; i < localStorage.length; i++) {
        const key = localStorage.key(i);
        if (key) {
          items[key] = localStorage.getItem(key) || '';
        }
      }
      return items;
    });

    return {
      cookies,
      localStorage,
      url: session.page.url()
    };
  }

  async destroySession(sessionId: string): Promise<void> {
    const session = this.sessions.get(sessionId);

    if (session) {
      await this.pool.release(session.page);
      this.sessions.delete(sessionId);
    }
  }

  cleanupExpiredSessions(): void {
    const now = Date.now();
    const expiredSessions: string[] = [];

    for (const [id, session] of this.sessions) {
      const age = now - session.lastAccessedAt.getTime();

      if (age > 3600000) { // 1 hour
        expiredSessions.push(id);
      }
    }

    expiredSessions.forEach(id => this.destroySession(id));
  }
}

interface BrowserSession {
  id: string;
  page: Page;
  createdAt: Date;
  lastAccessedAt: Date;
}

interface SessionState {
  cookies: any[];
  localStorage: Record<string, string>;
  url: string;
}
```

### Phase 4: Monitoring & Optimization (Week 2)

#### 4.1 Pool Metrics & Monitoring

**File**: `apps/api/src/services/browser-pool/PoolMetrics.ts`

```typescript
export class PoolMetrics {
  private acquisitions: number[] = [];
  private releases: number = 0;
  private errors: number = 0;
  private acquisitionTimes: number[] = [];
  private memorySnapshots: number[] = [];

  recordAcquisition(timeMs: number): void {
    this.acquisitions.push(Date.now());
    this.acquisitionTimes.push(timeMs);

    // Keep only last hour of data
    const oneHourAgo = Date.now() - 3600000;
    this.acquisitions = this.acquisitions.filter(t => t > oneHourAgo);
    this.acquisitionTimes = this.acquisitionTimes.slice(-1000);
  }

  recordRelease(): void {
    this.releases++;
  }

  recordError(): void {
    this.errors++;
  }

  recordMemoryUsage(memoryMB: number): void {
    this.memorySnapshots.push(memoryMB);
    this.memorySnapshots = this.memorySnapshots.slice(-100);
  }

  getSnapshot(): MetricsSnapshot {
    return {
      acquisitionRate: this.calculateRate(this.acquisitions),
      avgAcquisitionTime: this.average(this.acquisitionTimes),
      totalAcquisitions: this.acquisitions.length,
      totalReleases: this.releases,
      totalErrors: this.errors,
      avgMemoryUsage: this.average(this.memorySnapshots),
      peakMemoryUsage: Math.max(...this.memorySnapshots, 0)
    };
  }

  private calculateRate(timestamps: number[]): number {
    if (timestamps.length < 2) return 0;

    const duration = timestamps[timestamps.length - 1] - timestamps[0];
    if (duration === 0) return 0;

    return (timestamps.length / duration) * 1000; // Per second
  }

  private average(numbers: number[]): number {
    if (numbers.length === 0) return 0;
    return numbers.reduce((a, b) => a + b, 0) / numbers.length;
  }
}

interface MetricsSnapshot {
  acquisitionRate: number;
  avgAcquisitionTime: number;
  totalAcquisitions: number;
  totalReleases: number;
  totalErrors: number;
  avgMemoryUsage: number;
  peakMemoryUsage: number;
}
```

## Configuration Examples

```typescript
// Development configuration
const devConfig: PoolConfig = {
  minInstances: 1,
  maxInstances: 3,
  maxContextsPerBrowser: 2,
  maxPagesPerContext: 5,
  browserType: 'chromium',
  launchOptions: {
    headless: true,
    args: ['--no-sandbox', '--disable-dev-shm-usage']
  },
  contextOptions: {},
  recycleThreshold: 100,
  idleTimeoutMs: 60000,
  healthCheckIntervalMs: 30000,
  resourceLimits: {
    maxMemoryMB: 500,
    maxCPUPercent: 70
  }
};

// Production configuration
const prodConfig: PoolConfig = {
  minInstances: 5,
  maxInstances: 20,
  maxContextsPerBrowser: 5,
  maxPagesPerContext: 10,
  browserType: 'chromium',
  launchOptions: {
    headless: 'new',
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      '--disable-dev-shm-usage',
      '--disable-accelerated-2d-canvas',
      '--no-first-run',
      '--no-zygote',
      '--single-process',
      '--disable-gpu'
    ]
  },
  contextOptions: {
    viewport: { width: 1920, height: 1080 }
  },
  recycleThreshold: 1000,
  idleTimeoutMs: 300000,
  healthCheckIntervalMs: 60000,
  resourceLimits: {
    maxMemoryMB: 1000,
    maxCPUPercent: 80
  }
};
```

## Docker Optimization

```dockerfile
FROM node:20-slim

# Install dependencies for Chromium
RUN apt-get update && apt-get install -y \
    wget \
    ca-certificates \
    fonts-liberation \
    libappindicator3-1 \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libc6 \
    libcairo2 \
    libcups2 \
    libdbus-1-3 \
    libexpat1 \
    libfontconfig1 \
    libgbm1 \
    libgcc1 \
    libglib2.0-0 \
    libgtk-3-0 \
    libnspr4 \
    libnss3 \
    libpango-1.0-0 \
    libpangocairo-1.0-0 \
    libstdc++6 \
    libx11-6 \
    libx11-xcb1 \
    libxcb1 \
    libxcomposite1 \
    libxcursor1 \
    libxdamage1 \
    libxext6 \
    libxfixes3 \
    libxi6 \
    libxrandr2 \
    libxrender1 \
    libxss1 \
    libxtst6 \
    lsb-release \
    xdg-utils \
    && rm -rf /var/lib/apt/lists/*

# Create user for running browser
RUN groupadd -r pptruser && useradd -r -g pptruser -G audio,video pptruser \
    && mkdir -p /home/pptruser/Downloads \
    && chown -R pptruser:pptruser /home/pptruser

USER pptruser

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

CMD ["node", "dist/browser-pool-service.js"]
```

## Performance Benchmarks

### Expected Performance
- **Page Acquisition**: <100ms average
- **Memory per Browser**: 100-300MB
- **Pages per Browser**: 50-100
- **Concurrent Scrapers**: 100-500
- **Startup Time**: <5 seconds

### Resource Calculations
```
Total Memory = (Number of Browsers × 200MB) + (Number of Pages × 10MB) + 500MB overhead

Example for 10 browsers with 50 pages each:
= (10 × 200MB) + (500 × 10MB) + 500MB
= 2000MB + 5000MB + 500MB
= 7.5GB RAM
```

## Implementation Timeline

- **Week 1**: Core browser pool implementation
- **Week 2**: Auto-scaling and session management
- **Week 3**: Monitoring and optimization
- **Week 4**: Production deployment and tuning

## Success Criteria

1. **<100ms page acquisition time**
2. **Zero zombie processes**
3. **90% browser instance utilization**
4. **Automatic scaling response <10 seconds**
5. **Memory usage <300MB per browser**

## Dependencies

```json
{
  "playwright": "^1.48.0",
  "puppeteer-cluster": "^0.23.0",
  "generic-pool": "^3.9.0",
  "p-queue": "^8.0.1"
}
```