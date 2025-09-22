# Stealth Browser Automation Implementation Plan

## Executive Summary
Transform Firecrawl's basic Playwright service into a state-of-the-art stealth browser automation system using cutting-edge anti-detection technologies. This plan outlines the implementation of advanced browser fingerprinting bypass, behavioral mimicking, and detection evasion techniques.

## Current State Analysis
- **Location**: `apps/playwright-service-ts/api.ts`
- **Current Implementation**: Basic Playwright with user-agent rotation only
- **Weaknesses**:
  - No fingerprint spoofing
  - No WebDriver detection bypass
  - No canvas fingerprinting protection
  - No WebRTC leak prevention
  - Detectable automation markers

## Recommended Solution Stack

### Primary Option: Nodriver (Python)
**Package**: `nodriver`
**Version**: Latest (0.38+)
**Repository**: https://github.com/ultrafunkamsterdam/nodriver

#### Why Nodriver?
- Successor to undetected-chromedriver, purpose-built for stealth
- Direct CDP communication bypassing WebDriver entirely
- 25% success rate against major anti-bot services in 2025 tests
- Async-first architecture aligning with modern scraping needs
- Active development and community support

#### Installation:
```bash
pip install nodriver
```

### Alternative Option: Puppeteer-Extra with Stealth Plugin (Node.js)
**Packages**:
- `puppeteer-extra`: ^3.3.6
- `puppeteer-extra-plugin-stealth`: ^2.11.2
- `puppeteer-extra-plugin-recaptcha`: ^3.6.8

#### Installation:
```bash
npm install puppeteer-extra puppeteer-extra-plugin-stealth puppeteer-extra-plugin-recaptcha
```

### Playwright Enhancement: Rebrowser Patches
**Repository**: https://github.com/rebrowser/rebrowser-patches
**Note**: Community-maintained as of Feb 2025

## Implementation Architecture

### Phase 1: Core Stealth Engine (Week 1)

#### 1.1 Replace Current Playwright Service

**File**: Create new `apps/stealth-browser-service/index.ts`

```typescript
import { chromium } from 'playwright-extra';
import StealthPlugin from 'puppeteer-extra-plugin-stealth';

// Apply stealth patches
chromium.use(StealthPlugin());

export class StealthBrowserService {
  private browser: Browser;
  private contextPool: Map<string, BrowserContext> = new Map();

  async initialize() {
    // Launch with stealth configuration
    this.browser = await chromium.launch({
      headless: 'new', // Use new headless mode
      args: [
        '--disable-blink-features=AutomationControlled',
        '--disable-dev-shm-usage',
        '--disable-web-security',
        '--disable-features=IsolateOrigins,site-per-process',
        '--no-sandbox',
        '--disable-setuid-sandbox',
        '--disable-accelerated-2d-canvas',
        '--disable-gpu',
        '--window-size=1920,1080',
        '--start-maximized',
        '--ignore-certificate-errors',
        '--allow-running-insecure-content',
        '--disable-webgl',
        '--disable-webgl2',
        '--disable-3d-apis'
      ]
    });
  }
}
```

#### 1.2 Implement Fingerprint Randomization

**File**: `apps/stealth-browser-service/fingerprint.ts`

```typescript
export class FingerprintManager {
  private fingerprints = {
    viewport: [
      { width: 1920, height: 1080 },
      { width: 1366, height: 768 },
      { width: 1440, height: 900 },
      { width: 1536, height: 864 },
      { width: 1600, height: 900 },
      { width: 1920, height: 1200 }
    ],
    userAgents: [
      // Chrome 131 on Windows 11
      'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
      // Chrome 131 on macOS
      'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
      // Chrome 130 on Linux
      'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36'
    ],
    languages: ['en-US', 'en-GB', 'en-CA', 'en-AU'],
    platforms: ['Win32', 'MacIntel', 'Linux x86_64'],
    webglVendors: [
      'Intel Inc.',
      'NVIDIA Corporation',
      'AMD',
      'Apple Inc.'
    ],
    webglRenderers: [
      'Intel Iris OpenGL Engine',
      'NVIDIA GeForce GTX 1660 Ti',
      'AMD Radeon Pro 5500M',
      'Apple M1'
    ]
  };

  generateFingerprint() {
    return {
      viewport: this.randomChoice(this.fingerprints.viewport),
      userAgent: this.randomChoice(this.fingerprints.userAgents),
      language: this.randomChoice(this.fingerprints.languages),
      platform: this.randomChoice(this.fingerprints.platforms),
      webglVendor: this.randomChoice(this.fingerprints.webglVendors),
      webglRenderer: this.randomChoice(this.fingerprints.webglRenderers),
      timezone: this.generateTimezone(),
      canvas: this.generateCanvasNoise()
    };
  }

  private generateCanvasNoise() {
    // Add slight variations to canvas fingerprinting
    return {
      r: Math.random() * 0.01,
      g: Math.random() * 0.01,
      b: Math.random() * 0.01
    };
  }
}
```

### Phase 2: Behavioral Mimicking (Week 1-2)

#### 2.1 Human-like Mouse Movement

**File**: `apps/stealth-browser-service/behavior.ts`

```typescript
export class HumanBehavior {
  async simulateMouseMovement(page: Page, from: Point, to: Point) {
    const steps = this.generateBezierPath(from, to);

    for (const point of steps) {
      await page.mouse.move(point.x, point.y);
      await this.randomDelay(10, 25);
    }
  }

  private generateBezierPath(from: Point, to: Point): Point[] {
    // Implement Bézier curve for natural mouse movement
    const points: Point[] = [];
    const steps = 20 + Math.floor(Math.random() * 10);

    // Add control points for curve
    const cp1 = {
      x: from.x + (to.x - from.x) * 0.25 + (Math.random() - 0.5) * 100,
      y: from.y + (to.y - from.y) * 0.25 + (Math.random() - 0.5) * 100
    };

    const cp2 = {
      x: from.x + (to.x - from.x) * 0.75 + (Math.random() - 0.5) * 100,
      y: from.y + (to.y - from.y) * 0.75 + (Math.random() - 0.5) * 100
    };

    for (let i = 0; i <= steps; i++) {
      const t = i / steps;
      points.push(this.bezierPoint(from, cp1, cp2, to, t));
    }

    return points;
  }

  async simulateScrolling(page: Page) {
    const scrollHeight = await page.evaluate(() => document.body.scrollHeight);
    let currentPosition = 0;

    while (currentPosition < scrollHeight) {
      const scrollAmount = 100 + Math.floor(Math.random() * 300);
      await page.evaluate((amount) => window.scrollBy(0, amount), scrollAmount);
      currentPosition += scrollAmount;

      // Random pause while "reading"
      if (Math.random() < 0.3) {
        await this.randomDelay(1000, 3000);
      }

      await this.randomDelay(100, 500);
    }
  }

  async simulateReading(page: Page) {
    // Simulate time spent on page
    await this.randomDelay(3000, 8000);

    // Random mouse movements
    for (let i = 0; i < 3 + Math.floor(Math.random() * 5); i++) {
      const from = await this.getRandomPoint(page);
      const to = await this.getRandomPoint(page);
      await this.simulateMouseMovement(page, from, to);
      await this.randomDelay(500, 2000);
    }
  }

  private async randomDelay(min: number, max: number) {
    const delay = min + Math.random() * (max - min);
    await new Promise(resolve => setTimeout(resolve, delay));
  }
}
```

#### 2.2 Advanced Detection Bypass

**File**: `apps/stealth-browser-service/bypass.ts`

```typescript
export class DetectionBypass {
  async applyEvasions(page: Page) {
    // Override navigator.webdriver
    await page.evaluateOnNewDocument(() => {
      Object.defineProperty(navigator, 'webdriver', {
        get: () => undefined
      });
    });

    // Mock Chrome runtime
    await page.evaluateOnNewDocument(() => {
      window.chrome = {
        runtime: {
          connect: () => {},
          sendMessage: () => {},
          onMessage: { addListener: () => {} }
        },
        loadTimes: () => {},
        csi: () => {}
      };
    });

    // Override permissions API
    await page.evaluateOnNewDocument(() => {
      const originalQuery = window.navigator.permissions.query;
      window.navigator.permissions.query = (parameters) => {
        if (parameters.name === 'notifications') {
          return Promise.resolve({ state: 'prompt' });
        }
        return originalQuery(parameters);
      };
    });

    // Hide automation indicators
    await page.evaluateOnNewDocument(() => {
      // Remove Playwright/Puppeteer specific properties
      delete window.__playwright;
      delete window.__puppeteer;
      delete window.Buffer;
      delete window.emit;

      // Override plugins to look realistic
      Object.defineProperty(navigator, 'plugins', {
        get: () => [
          { name: 'Chrome PDF Viewer', filename: 'internal-pdf-viewer' },
          { name: 'Native Client', filename: 'internal-nacl-plugin' },
          { name: 'Chrome PDF Plugin', filename: 'internal-pdf-viewer' }
        ]
      });

      // Mock realistic battery API
      Object.defineProperty(navigator, 'getBattery', {
        value: () => Promise.resolve({
          charging: true,
          chargingTime: 0,
          dischargingTime: Infinity,
          level: 0.87
        })
      });
    });

    // WebRTC leak prevention
    await page.evaluateOnNewDocument(() => {
      const rtc = window.RTCPeerConnection;
      window.RTCPeerConnection = function(...args) {
        const pc = new rtc(...args);
        pc.createDataChannel = () => ({});
        pc.createOffer = () => Promise.resolve({});
        return pc;
      };
      window.RTCPeerConnection.prototype = rtc.prototype;
    });

    // Canvas fingerprinting noise
    await page.evaluateOnNewDocument((noise) => {
      const originalToDataURL = HTMLCanvasElement.prototype.toDataURL;
      HTMLCanvasElement.prototype.toDataURL = function(...args) {
        const context = this.getContext('2d');
        if (context) {
          const imageData = context.getImageData(0, 0, this.width, this.height);
          for (let i = 0; i < imageData.data.length; i += 4) {
            imageData.data[i] += noise.r;
            imageData.data[i + 1] += noise.g;
            imageData.data[i + 2] += noise.b;
          }
          context.putImageData(imageData, 0, 0);
        }
        return originalToDataURL.apply(this, args);
      };
    }, this.fingerprint.canvas);
  }
}
```

### Phase 3: Integration with Existing System (Week 2)

#### 3.1 Update Scraper Engine Interface

**File**: `apps/api/src/scraper/scrapeURL/engines/stealth-browser/index.ts`

```typescript
import { StealthBrowserService } from '../../../../stealth-browser-service';
import { EngineScrapeResult } from '..';
import { Meta } from '../..';

export async function scrapeURLWithStealthBrowser(
  meta: Meta,
): Promise<EngineScrapeResult> {
  const service = new StealthBrowserService();
  const behavior = new HumanBehavior();

  try {
    const context = await service.createStealthContext(meta.options);
    const page = await context.newPage();

    // Apply all evasions
    await service.applyEvasions(page);

    // Navigate with human-like behavior
    await behavior.simulateReading(page);
    const response = await page.goto(meta.url, {
      waitUntil: 'networkidle',
      timeout: meta.abort.scrapeTimeout()
    });

    // Simulate human interaction
    await behavior.simulateScrolling(page);
    await behavior.simulateReading(page);

    // Extract content
    const html = await page.content();
    const statusCode = response?.status() || 200;

    return {
      url: meta.url,
      html,
      statusCode,
      proxyUsed: meta.options.proxy || 'none',
      engine: 'stealth-browser'
    };
  } finally {
    await service.cleanup();
  }
}
```

## Testing & Validation

### Detection Test Suite

1. **Fingerprint Tests**
   - https://bot.sannysoft.com/
   - https://browserleaks.com/javascript
   - https://pixelscan.net/
   - https://abrahamjuliot.github.io/creepjs/

2. **CAPTCHA Tests**
   - Cloudflare protected sites
   - DataDome protected sites
   - PerimeterX protected sites
   - Kasada protected sites

3. **Performance Metrics**
   - Memory usage per browser instance
   - Request success rate
   - Average page load time
   - CAPTCHA encounter rate

## Monitoring & Maintenance

### Key Metrics
- Detection rate by domain
- Browser crash frequency
- Memory leak detection
- Fingerprint uniqueness score

### Update Schedule
- Weekly: Update user agent strings
- Monthly: Review detection test results
- Quarterly: Update fingerprint database
- As needed: Patch new detection methods

## Risk Mitigation

1. **Fallback Strategy**: Maintain current Playwright service as fallback
2. **Gradual Rollout**: Deploy to 10% → 25% → 50% → 100% of traffic
3. **Detection Monitoring**: Alert on sudden spike in CAPTCHAs
4. **Legal Compliance**: Ensure robots.txt compliance remains enforced

## Cost Analysis

### Development Time
- Phase 1: 5 days
- Phase 2: 5 days
- Phase 3: 3 days
- Testing: 2 days
- **Total**: 15 days

### Infrastructure Impact
- Memory: +20% per browser instance
- CPU: +15% for behavior simulation
- Storage: Minimal impact

### Expected ROI
- 70% reduction in CAPTCHA encounters
- 85% improvement in scraping success rate
- 60% reduction in IP bans

## Dependencies

### NPM Packages
```json
{
  "puppeteer-extra": "^3.3.6",
  "puppeteer-extra-plugin-stealth": "^2.11.2",
  "puppeteer-extra-plugin-recaptcha": "^3.6.8",
  "playwright-extra": "^4.3.6",
  "random-useragent": "^0.5.0",
  "@devicefarmer/adbkit": "^3.2.5"
}
```

### Python Packages (if using Nodriver)
```requirements.txt
nodriver>=0.38
selenium-driverless>=1.8
undetected-chromedriver>=3.5.5
```

## Success Criteria

1. Pass rate >80% on bot detection test suites
2. CAPTCHA encounter rate <5% on average sites
3. Zero detectable automation properties in navigator
4. Maintain current scraping speed within 10% margin
5. Memory usage increase <25%

## Timeline

- **Week 1**: Core stealth engine implementation
- **Week 2**: Behavioral mimicking and advanced bypass
- **Week 3**: Integration and testing
- **Week 4**: Monitoring setup and production rollout

## Next Steps

1. Review and approve implementation plan
2. Set up development environment
3. Create feature branch for stealth browser implementation
4. Begin Phase 1 development
5. Establish testing benchmarks