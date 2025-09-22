# CAPTCHA Detection and Solving Implementation Plan

## Executive Summary
Implement an intelligent CAPTCHA detection and solving system that automatically identifies, classifies, and solves various CAPTCHA types using AI-powered services and smart retry strategies. This plan covers integration with leading CAPTCHA solving services and automated detection mechanisms.

## Current State Analysis
- **Current Implementation**: No CAPTCHA handling
- **Impact**: Complete failure when encountering CAPTCHAs
- **User Experience**: Manual intervention required
- **Success Rate**: 0% on CAPTCHA-protected sites

## CAPTCHA Types & Detection

### Common CAPTCHA Types (2025)
1. **reCAPTCHA v2**: "I'm not a robot" checkbox
2. **reCAPTCHA v3**: Score-based, invisible
3. **reCAPTCHA Enterprise**: Advanced risk analysis
4. **hCaptcha**: Privacy-focused alternative
5. **FunCaptcha (Arkose Labs)**: Image rotation puzzles
6. **GeeTest**: Sliding puzzles
7. **Cloudflare Turnstile**: Cloudflare's challenge
8. **DataDome**: Behavioral analysis
9. **PerimeterX/HUMAN**: Advanced bot detection
10. **AWS WAF CAPTCHA**: Amazon's solution

## Recommended CAPTCHA Solving Services

### Tier 1: AI-Powered Solutions

#### 1. CapSolver (Primary Recommendation)
- **Pricing**: $0.8 per 1000 reCAPTCHA v2
- **Speed**: ~5 seconds average
- **Success Rate**: 99%
- **API**: RESTful + WebSocket
- **Features**:
  - AI/ML-based solving
  - Chrome/Firefox extensions
  - All major CAPTCHA types

#### 2. 2Captcha
- **Pricing**: $1.0 per 1000 CAPTCHAs
- **Speed**: ~9 seconds for reCAPTCHA v2
- **Success Rate**: 95%+
- **Features**:
  - Human + AI hybrid
  - 15+ years experience
  - Success-based billing

#### 3. Anti-Captcha
- **Pricing**: $0.7 per 1000 CAPTCHAs
- **Speed**: ~7 seconds average
- **Success Rate**: 99%
- **Features**:
  - Since 2007
  - Global solver network
  - Plugin support

## Implementation Architecture

### Phase 1: Core CAPTCHA Detection (Week 1)

#### 1.1 CAPTCHA Detector Service

**File**: `apps/api/src/services/captcha/CaptchaDetector.ts`

```typescript
export interface CaptchaInfo {
  type: 'recaptcha-v2' | 'recaptcha-v3' | 'hcaptcha' | 'funcaptcha' |
        'geetest' | 'turnstile' | 'datadome' | 'perimeterx' | 'aws-waf' | 'unknown';
  siteKey?: string;
  pageUrl: string;
  action?: string;
  dataS?: string;
  apiDomain?: string;
  detected: boolean;
  confidence: number;
  metadata: Record<string, any>;
}

export class CaptchaDetector {
  private patterns = {
    'recaptcha-v2': {
      selectors: [
        'div.g-recaptcha',
        'iframe[src*="google.com/recaptcha"]',
        'iframe[title="reCAPTCHA"]'
      ],
      scripts: [
        'google.com/recaptcha/api.js',
        'google.com/recaptcha/enterprise.js',
        'recaptcha.net/recaptcha/api.js'
      ],
      dataAttributes: ['data-sitekey', 'data-callback']
    },
    'recaptcha-v3': {
      selectors: [
        '.grecaptcha-badge'
      ],
      scripts: [
        'google.com/recaptcha/api.js?render=',
        'google.com/recaptcha/enterprise.js?render='
      ],
      globalVars: ['grecaptcha']
    },
    'hcaptcha': {
      selectors: [
        'div.h-captcha',
        'iframe[src*="hcaptcha.com"]'
      ],
      scripts: [
        'hcaptcha.com/1/api.js',
        'js.hcaptcha.com'
      ],
      dataAttributes: ['data-sitekey', 'data-callback']
    },
    'funcaptcha': {
      selectors: [
        'div[id*="funcaptcha"]',
        'iframe[src*="funcaptcha.com"]',
        'iframe[src*="arkoselabs.com"]'
      ],
      scripts: [
        'funcaptcha.com',
        'arkoselabs.com'
      ],
      dataAttributes: ['data-pkey']
    },
    'geetest': {
      selectors: [
        '.geetest_holder',
        'div[id*="geetest"]'
      ],
      scripts: [
        'static.geetest.com',
        'api.geetest.com'
      ]
    },
    'turnstile': {
      selectors: [
        '.cf-turnstile',
        'iframe[src*="challenges.cloudflare.com"]'
      ],
      scripts: [
        'challenges.cloudflare.com/turnstile'
      ],
      dataAttributes: ['data-sitekey', 'data-callback']
    },
    'datadome': {
      selectors: [
        '.datadome-captcha',
        'script[src*="datadome.co"]'
      ],
      scripts: [
        'ct.datadome.co',
        'js.datadome.co'
      ],
      cookies: ['datadome']
    },
    'perimeterx': {
      selectors: [],
      scripts: [
        'client.perimeterx.net',
        'px-cdn.net'
      ],
      cookies: ['_px', '_pxvid']
    }
  };

  async detect(page: Page | string): Promise<CaptchaInfo> {
    let html: string;
    let url: string;

    if (typeof page === 'string') {
      html = page;
      url = '';
    } else {
      html = await page.content();
      url = page.url();

      // Also check for dynamic CAPTCHAs
      await this.checkDynamicCaptchas(page);
    }

    for (const [type, pattern] of Object.entries(this.patterns)) {
      const result = await this.checkPattern(html, pattern, type as any);
      if (result.detected) {
        result.pageUrl = url;
        return result;
      }
    }

    // Check for generic CAPTCHA indicators
    const genericDetected = await this.detectGenericCaptcha(html);
    if (genericDetected) {
      return {
        type: 'unknown',
        pageUrl: url,
        detected: true,
        confidence: 0.5,
        metadata: {}
      };
    }

    return {
      type: 'unknown',
      pageUrl: url,
      detected: false,
      confidence: 0,
      metadata: {}
    };
  }

  private async checkPattern(
    html: string,
    pattern: any,
    type: CaptchaInfo['type']
  ): Promise<CaptchaInfo> {
    let detected = false;
    let confidence = 0;
    const metadata: Record<string, any> = {};

    // Check selectors
    if (pattern.selectors) {
      for (const selector of pattern.selectors) {
        if (html.includes(selector.replace(/[\[\].*]/g, ''))) {
          detected = true;
          confidence += 0.3;
        }
      }
    }

    // Check scripts
    if (pattern.scripts) {
      for (const script of pattern.scripts) {
        if (html.includes(script)) {
          detected = true;
          confidence += 0.4;

          // Extract site key from script URL if present
          const siteKeyMatch = html.match(new RegExp(`${script}.*?[?&]render=([^&"'\\s]+)`));
          if (siteKeyMatch) {
            metadata.siteKey = siteKeyMatch[1];
          }
        }
      }
    }

    // Check data attributes
    if (pattern.dataAttributes) {
      for (const attr of pattern.dataAttributes) {
        const match = html.match(new RegExp(`${attr}="([^"]+)"`));
        if (match) {
          detected = true;
          confidence += 0.3;
          metadata[attr.replace('data-', '')] = match[1];
        }
      }
    }

    return {
      type,
      detected,
      confidence: Math.min(confidence, 1),
      pageUrl: '',
      metadata
    };
  }

  private async checkDynamicCaptchas(page: Page): Promise<void> {
    // Check for dynamically loaded CAPTCHAs
    try {
      await page.evaluate(() => {
        // Check for global CAPTCHA objects
        return {
          recaptcha: typeof (window as any).grecaptcha !== 'undefined',
          hcaptcha: typeof (window as any).hcaptcha !== 'undefined',
          turnstile: typeof (window as any).turnstile !== 'undefined'
        };
      });
    } catch (e) {
      // Page might be navigating
    }
  }

  private async detectGenericCaptcha(html: string): Promise<boolean> {
    const genericIndicators = [
      'captcha',
      'challenge-form',
      'verify-human',
      'bot-check',
      'security-check',
      'access-denied',
      'please verify',
      'confirm you are human'
    ];

    const lowerHtml = html.toLowerCase();
    return genericIndicators.some(indicator => lowerHtml.includes(indicator));
  }
}
```

### Phase 2: CAPTCHA Solving Integration (Week 1-2)

#### 2.1 Multi-Service Solver Manager

**File**: `apps/api/src/services/captcha/CaptchaSolver.ts`

```typescript
import { CapSolverClient } from './providers/CapSolverClient';
import { TwoCaptchaClient } from './providers/TwoCaptchaClient';
import { AntiCaptchaClient } from './providers/AntiCaptchaClient';

export interface SolveOptions {
  type: string;
  siteKey?: string;
  pageUrl: string;
  action?: string;
  minScore?: number;
  proxy?: string;
  userAgent?: string;
  cookies?: string;
  apiDomain?: string;
  enterprise?: boolean;
  invisible?: boolean;
}

export interface SolveResult {
  solution: string;
  cost: number;
  solvingTime: number;
  service: string;
  taskId?: string;
}

export class CaptchaSolver {
  private services: Map<string, CaptchaService> = new Map();
  private primaryService: string;
  private metrics: Map<string, ServiceMetrics> = new Map();

  constructor() {
    this.initializeServices();
    this.primaryService = process.env.PRIMARY_CAPTCHA_SERVICE || 'capsolver';
  }

  private initializeServices() {
    // Initialize CapSolver
    if (process.env.CAPSOLVER_API_KEY) {
      this.services.set('capsolver', new CapSolverClient(
        process.env.CAPSOLVER_API_KEY
      ));
    }

    // Initialize 2Captcha
    if (process.env.TWOCAPTCHA_API_KEY) {
      this.services.set('2captcha', new TwoCaptchaClient(
        process.env.TWOCAPTCHA_API_KEY
      ));
    }

    // Initialize Anti-Captcha
    if (process.env.ANTICAPTCHA_API_KEY) {
      this.services.set('anticaptcha', new AntiCaptchaClient(
        process.env.ANTICAPTCHA_API_KEY
      ));
    }
  }

  async solve(options: SolveOptions): Promise<SolveResult> {
    const startTime = Date.now();

    // Try primary service first
    try {
      const result = await this.solveWithService(this.primaryService, options);
      this.recordMetrics(this.primaryService, true, Date.now() - startTime);
      return result;
    } catch (primaryError) {
      logger.warn('Primary CAPTCHA service failed', {
        service: this.primaryService,
        error: primaryError
      });

      this.recordMetrics(this.primaryService, false, Date.now() - startTime);
    }

    // Fallback to other services
    for (const [serviceName, service] of this.services.entries()) {
      if (serviceName === this.primaryService) continue;

      try {
        const result = await this.solveWithService(serviceName, options);
        this.recordMetrics(serviceName, true, Date.now() - startTime);
        return result;
      } catch (error) {
        logger.warn('Fallback CAPTCHA service failed', {
          service: serviceName,
          error
        });
        this.recordMetrics(serviceName, false, Date.now() - startTime);
      }
    }

    throw new Error('All CAPTCHA solving services failed');
  }

  private async solveWithService(
    serviceName: string,
    options: SolveOptions
  ): Promise<SolveResult> {
    const service = this.services.get(serviceName);
    if (!service) {
      throw new Error(`Service ${serviceName} not configured`);
    }

    return service.solve(options);
  }

  private recordMetrics(service: string, success: boolean, time: number) {
    if (!this.metrics.has(service)) {
      this.metrics.set(service, {
        totalAttempts: 0,
        successfulAttempts: 0,
        totalTime: 0,
        avgTime: 0,
        successRate: 0
      });
    }

    const metrics = this.metrics.get(service)!;
    metrics.totalAttempts++;
    metrics.totalTime += time;

    if (success) {
      metrics.successfulAttempts++;
    }

    metrics.avgTime = metrics.totalTime / metrics.totalAttempts;
    metrics.successRate = metrics.successfulAttempts / metrics.totalAttempts;
  }

  async getBalance(): Promise<Map<string, number>> {
    const balances = new Map<string, number>();

    for (const [name, service] of this.services.entries()) {
      try {
        const balance = await service.getBalance();
        balances.set(name, balance);
      } catch (e) {
        balances.set(name, -1);
      }
    }

    return balances;
  }
}
```

#### 2.2 CapSolver Integration

**File**: `apps/api/src/services/captcha/providers/CapSolverClient.ts`

```typescript
export class CapSolverClient implements CaptchaService {
  private apiKey: string;
  private baseUrl = 'https://api.capsolver.com';

  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }

  async solve(options: SolveOptions): Promise<SolveResult> {
    // Create task based on CAPTCHA type
    const task = this.buildTask(options);

    // Submit task
    const createResponse = await fetch(`${this.baseUrl}/createTask`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        clientKey: this.apiKey,
        task,
        languagePool: 'en',
        callbackUrl: process.env.CAPTCHA_CALLBACK_URL
      })
    });

    if (!createResponse.ok) {
      throw new Error('Failed to create CAPTCHA task');
    }

    const { taskId } = await createResponse.json();

    // Poll for result
    const result = await this.pollResult(taskId);

    return {
      solution: result.solution.gRecaptchaResponse || result.solution.token,
      cost: result.cost || 0.001,
      solvingTime: result.solvingTime || 0,
      service: 'capsolver',
      taskId
    };
  }

  private buildTask(options: SolveOptions): any {
    const taskMap: Record<string, any> = {
      'recaptcha-v2': {
        type: 'ReCaptchaV2TaskProxyLess',
        websiteURL: options.pageUrl,
        websiteKey: options.siteKey,
        isInvisible: options.invisible || false
      },
      'recaptcha-v3': {
        type: 'ReCaptchaV3TaskProxyLess',
        websiteURL: options.pageUrl,
        websiteKey: options.siteKey,
        pageAction: options.action || 'submit',
        minScore: options.minScore || 0.7
      },
      'hcaptcha': {
        type: 'HCaptchaTaskProxyLess',
        websiteURL: options.pageUrl,
        websiteKey: options.siteKey
      },
      'funcaptcha': {
        type: 'FunCaptchaTaskProxyLess',
        websiteURL: options.pageUrl,
        websitePublicKey: options.siteKey,
        data: options.apiDomain ? { blob: options.apiDomain } : undefined
      },
      'turnstile': {
        type: 'AntiTurnstileTaskProxyLess',
        websiteURL: options.pageUrl,
        websiteKey: options.siteKey
      },
      'geetest': {
        type: 'GeeTestTaskProxyLess',
        websiteURL: options.pageUrl,
        gt: options.siteKey,
        challenge: options.action
      }
    };

    const task = taskMap[options.type];

    if (!task) {
      throw new Error(`Unsupported CAPTCHA type: ${options.type}`);
    }

    // Add proxy if provided
    if (options.proxy) {
      task.type = task.type.replace('ProxyLess', '');
      const proxyParts = options.proxy.match(/^(https?):\/\/([^:]+):([^@]+)@([^:]+):(\d+)$/);
      if (proxyParts) {
        task.proxyType = proxyParts[1];
        task.proxyAddress = proxyParts[4];
        task.proxyPort = parseInt(proxyParts[5]);
        task.proxyLogin = proxyParts[2];
        task.proxyPassword = proxyParts[3];
      }
    }

    if (options.userAgent) {
      task.userAgent = options.userAgent;
    }

    if (options.cookies) {
      task.cookies = options.cookies;
    }

    return task;
  }

  private async pollResult(taskId: string, maxAttempts = 120): Promise<any> {
    for (let i = 0; i < maxAttempts; i++) {
      await new Promise(resolve => setTimeout(resolve, 2000));

      const response = await fetch(`${this.baseUrl}/getTaskResult`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          clientKey: this.apiKey,
          taskId
        })
      });

      const result = await response.json();

      if (result.status === 'ready') {
        return result;
      }

      if (result.status === 'failed') {
        throw new Error(`CAPTCHA solving failed: ${result.errorDescription}`);
      }
    }

    throw new Error('CAPTCHA solving timeout');
  }

  async getBalance(): Promise<number> {
    const response = await fetch(`${this.baseUrl}/getBalance`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        clientKey: this.apiKey
      })
    });

    const result = await response.json();
    return result.balance;
  }
}
```

### Phase 3: Intelligent CAPTCHA Handling (Week 2)

#### 3.1 Auto-Solve Integration with Scraper

**File**: `apps/api/src/scraper/scrapeURL/middleware/captchaMiddleware.ts`

```typescript
export class CaptchaMiddleware {
  private detector: CaptchaDetector;
  private solver: CaptchaSolver;
  private cache: Map<string, CachedSolution> = new Map();

  constructor() {
    this.detector = new CaptchaDetector();
    this.solver = new CaptchaSolver();
  }

  async handlePage(page: Page, meta: Meta): Promise<boolean> {
    // Detect CAPTCHA
    const captchaInfo = await this.detector.detect(page);

    if (!captchaInfo.detected) {
      return false; // No CAPTCHA detected
    }

    logger.info('CAPTCHA detected', {
      type: captchaInfo.type,
      url: meta.url,
      confidence: captchaInfo.confidence
    });

    // Check cache for recent solution
    const cached = this.getCachedSolution(meta.url, captchaInfo);
    if (cached) {
      await this.applySolution(page, captchaInfo, cached.solution);
      return true;
    }

    // Solve CAPTCHA
    try {
      const solution = await this.solver.solve({
        type: captchaInfo.type,
        siteKey: captchaInfo.metadata.sitekey,
        pageUrl: meta.url,
        proxy: meta.proxyUsed?.url,
        userAgent: await page.evaluate(() => navigator.userAgent),
        cookies: await this.extractCookies(page)
      });

      // Apply solution
      await this.applySolution(page, captchaInfo, solution.solution);

      // Cache solution
      this.cacheSolution(meta.url, captchaInfo, solution);

      // Track metrics
      meta.costTracking.addCost('captcha', solution.cost);

      logger.info('CAPTCHA solved successfully', {
        type: captchaInfo.type,
        service: solution.service,
        time: solution.solvingTime,
        cost: solution.cost
      });

      return true;
    } catch (error) {
      logger.error('CAPTCHA solving failed', {
        error,
        type: captchaInfo.type,
        url: meta.url
      });

      throw new CaptchaSolvingError(captchaInfo.type);
    }
  }

  private async applySolution(
    page: Page,
    captchaInfo: CaptchaInfo,
    solution: string
  ): Promise<void> {
    switch (captchaInfo.type) {
      case 'recaptcha-v2':
        await this.applyRecaptchaV2Solution(page, solution);
        break;

      case 'recaptcha-v3':
        await this.applyRecaptchaV3Solution(page, solution);
        break;

      case 'hcaptcha':
        await this.applyHCaptchaSolution(page, solution);
        break;

      case 'turnstile':
        await this.applyTurnstileSolution(page, solution);
        break;

      default:
        throw new Error(`Unsupported CAPTCHA type for solution application: ${captchaInfo.type}`);
    }
  }

  private async applyRecaptchaV2Solution(page: Page, token: string): Promise<void> {
    await page.evaluate((token) => {
      // Find callback function
      const callback = (window as any).___grecaptcha_cfg?.clients?.[0]?.callback;

      if (callback) {
        // Call callback with token
        if (typeof callback === 'string') {
          (window as any)[callback](token);
        } else {
          callback(token);
        }
      } else {
        // Fallback: inject token into response field
        const textarea = document.querySelector('#g-recaptcha-response');
        if (textarea) {
          (textarea as HTMLTextAreaElement).value = token;
        }

        // Try to submit form
        const form = document.querySelector('form');
        if (form) {
          form.submit();
        }
      }
    }, token);

    // Wait for navigation or content change
    await Promise.race([
      page.waitForNavigation({ waitUntil: 'networkidle' }),
      page.waitForTimeout(5000)
    ]);
  }

  private async applyRecaptchaV3Solution(page: Page, token: string): Promise<void> {
    await page.evaluate((token) => {
      // reCAPTCHA v3 tokens are usually submitted via form or AJAX
      const tokenInput = document.querySelector('input[name="g-recaptcha-response"]');
      if (tokenInput) {
        (tokenInput as HTMLInputElement).value = token;
      }

      // Trigger form submission if exists
      const form = tokenInput?.closest('form');
      if (form) {
        form.submit();
      }
    }, token);
  }

  private async applyHCaptchaSolution(page: Page, token: string): Promise<void> {
    await page.evaluate((token) => {
      // Set token in response field
      const responseInput = document.querySelector('[name="h-captcha-response"]');
      if (responseInput) {
        (responseInput as HTMLInputElement).value = token;
      }

      // Trigger callback
      const callback = (window as any).hcaptchaCallback;
      if (callback) {
        callback(token);
      }
    }, token);
  }

  private async applyTurnstileSolution(page: Page, token: string): Promise<void> {
    await page.evaluate((token) => {
      const responseInput = document.querySelector('[name="cf-turnstile-response"]');
      if (responseInput) {
        (responseInput as HTMLInputElement).value = token;
      }

      // Trigger callback if exists
      const callback = (window as any).turnstileCallback;
      if (callback) {
        callback(token);
      }
    }, token);
  }

  private getCachedSolution(
    url: string,
    captchaInfo: CaptchaInfo
  ): CachedSolution | null {
    const key = `${new URL(url).hostname}:${captchaInfo.type}:${captchaInfo.metadata.sitekey}`;
    const cached = this.cache.get(key);

    if (cached && cached.expiresAt > Date.now()) {
      return cached;
    }

    return null;
  }

  private cacheSolution(
    url: string,
    captchaInfo: CaptchaInfo,
    solution: SolveResult
  ): void {
    const key = `${new URL(url).hostname}:${captchaInfo.type}:${captchaInfo.metadata.sitekey}`;

    this.cache.set(key, {
      solution: solution.solution,
      expiresAt: Date.now() + 120000 // 2 minutes
    });

    // Clean old entries
    if (this.cache.size > 100) {
      const entries = Array.from(this.cache.entries());
      const expired = entries.filter(([_, v]) => v.expiresAt < Date.now());
      expired.forEach(([k]) => this.cache.delete(k));
    }
  }
}
```

## Testing & Validation

### Test Sites
```typescript
const testSites = {
  'recaptcha-v2': [
    'https://www.google.com/recaptcha/api2/demo',
    'https://patrickhlauke.github.io/recaptcha/'
  ],
  'recaptcha-v3': [
    'https://recaptcha-demo.appspot.com/recaptcha-v3-request-scores.php'
  ],
  'hcaptcha': [
    'https://accounts.hcaptcha.com/demo'
  ],
  'cloudflare': [
    'https://nowsecure.nl/',
    'https://bot.sannysoft.com/'
  ]
};
```

### Performance Metrics
- Average solving time per CAPTCHA type
- Success rate by service
- Cost per successful solve
- Cache hit rate
- Detection accuracy

## Monitoring Dashboard

```typescript
interface CaptchaMetrics {
  totalEncountered: number;
  totalSolved: number;
  totalFailed: number;
  byType: Map<string, TypeMetrics>;
  byService: Map<string, ServiceMetrics>;
  totalCost: number;
  avgSolveTime: number;
}

interface TypeMetrics {
  encountered: number;
  solved: number;
  failed: number;
  avgSolveTime: number;
}
```

## Cost Optimization

### Strategies
1. **Intelligent Service Selection**: Use cheaper services for easier CAPTCHAs
2. **Solution Caching**: Reuse tokens within validity period
3. **Batch Solving**: Pre-solve CAPTCHAs for known protected sites
4. **Threshold Management**: Skip low-value pages with CAPTCHAs

### Budget Controls
```typescript
class CaptchaBudgetManager {
  private dailyLimit: number = 10.00; // $10 per day
  private spent: number = 0;

  canSolve(estimatedCost: number): boolean {
    return this.spent + estimatedCost <= this.dailyLimit;
  }

  recordSpend(amount: number): void {
    this.spent += amount;
  }
}
```

## Implementation Timeline

- **Week 1**: CAPTCHA detection system
- **Week 2**: Service integrations (CapSolver, 2Captcha, Anti-Captcha)
- **Week 3**: Auto-solve middleware and testing
- **Week 4**: Monitoring and optimization

## Success Criteria

1. **Detection Rate**: >95% accurate CAPTCHA detection
2. **Solve Rate**: >90% successful solving
3. **Speed**: <10 seconds average solve time
4. **Cost**: <$0.002 per CAPTCHA average
5. **Automation**: Zero manual intervention required

## Dependencies

```json
{
  "capsolver-npm": "^1.0.0",
  "2captcha": "^3.0.7",
  "anticaptcha": "^2.0.0",
  "@antiadmin/anticaptchaofficial": "^1.0.45",
  "puppeteer-extra-plugin-recaptcha": "^3.6.8"
}
```

## Environment Variables

```env
# Primary service
PRIMARY_CAPTCHA_SERVICE=capsolver

# CapSolver
CAPSOLVER_API_KEY=your_api_key

# 2Captcha
TWOCAPTCHA_API_KEY=your_api_key

# Anti-Captcha
ANTICAPTCHA_API_KEY=your_api_key

# Budget limits
CAPTCHA_DAILY_BUDGET=10.00
CAPTCHA_PER_REQUEST_LIMIT=0.01

# Callback URL for async solving
CAPTCHA_CALLBACK_URL=https://api.yoursite.com/captcha/callback
```