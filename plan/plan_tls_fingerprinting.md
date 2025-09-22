# TLS Fingerprinting Bypass Implementation Plan

## Executive Summary
Implement advanced TLS fingerprinting bypass techniques to evade JA3/JA4 detection and mimic real browser TLS signatures. This plan details the integration of curl-impersonate, tls-client, and custom TLS configuration to defeat sophisticated bot detection systems.

## Current State Analysis
- **Location**: `apps/api/src/scraper/scrapeURL/engines/fetch/index.ts`
- **Current Implementation**:
  - Standard Node.js undici fetch
  - Default TLS handshake (easily detectable)
  - No JA3/JA4 fingerprint randomization
  - No cipher suite customization
  - Immediately identifiable as Node.js client

## Technical Background

### What is TLS Fingerprinting?
TLS fingerprinting creates a unique identifier from the TLS ClientHello message, including:
- TLS version
- Cipher suites offered
- Extensions and their order
- Elliptic curves
- Signature algorithms

### JA3 vs JA4
- **JA3**: MD5 hash of TLS parameters (widely used but becoming outdated)
- **JA4**: Enhanced format with better accuracy for TLS 1.3 and QUIC
- **JA4+**: Extended suite including JA4H (HTTP), JA4S (server), JA4T (TCP)

## Recommended Solution Stack

### Primary Solution: curl_cffi (Python)

**Package**: `curl_cffi`
**Version**: 0.7.3+ (Latest as of 2025)
**Repository**: https://github.com/lexiforest/curl_cffi

#### Why curl_cffi?
- Wraps curl-impersonate with Python bindings
- Supports 15+ browser fingerprints out of the box
- Dynamic JA3 rotation (changes per request for Chrome 110+)
- Proven effectiveness against Cloudflare, DataDome, Akamai
- Active maintenance and community support

#### Installation:
```bash
pip install curl-cffi[speedups]
```

### Alternative Solution: tls-client (Go/Node.js)

**Package**: `tls-client` (Go) / `node-tls-client` (Node.js wrapper)
**Repository**: https://github.com/bogdanfinn/tls-client

#### Installation:
```bash
npm install tls-client
```

## Implementation Architecture

### Phase 1: Core TLS Engine (Week 1)

#### 1.1 Create Python TLS Service

**File**: `apps/tls-service/main.py`

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from curl_cffi import requests
from typing import Optional, Dict, Any, List
import random
import json
import asyncio
from datetime import datetime, timedelta

app = FastAPI()

class TLSRequest(BaseModel):
    url: str
    method: str = "GET"
    headers: Optional[Dict[str, str]] = None
    data: Optional[str] = None
    json_data: Optional[Dict[str, Any]] = None
    impersonate: Optional[str] = None
    proxy: Optional[str] = None
    timeout: int = 30
    follow_redirects: bool = True
    cookies: Optional[Dict[str, str]] = None

class TLSResponse(BaseModel):
    status_code: int
    headers: Dict[str, str]
    content: str
    cookies: Dict[str, str]
    url: str
    ja3_fingerprint: str
    impersonation_used: str

class FingerprintRotator:
    """Manages browser fingerprint rotation"""

    # Latest browser versions as of 2025
    BROWSER_VERSIONS = {
        "chrome": [
            "chrome131",  # Latest stable
            "chrome130",
            "chrome129",
            "chrome128",
            "chrome127",
            "chrome126"
        ],
        "firefox": [
            "firefox128",
            "firefox127",
            "firefox126"
        ],
        "safari": [
            "safari18",
            "safari17.5",
            "safari17"
        ],
        "edge": [
            "edge131",
            "edge130",
            "edge129"
        ]
    }

    # Weighted selection based on market share
    BROWSER_WEIGHTS = {
        "chrome": 0.65,    # 65% market share
        "firefox": 0.10,   # 10% market share
        "safari": 0.20,    # 20% market share
        "edge": 0.05       # 5% market share
    }

    def __init__(self):
        self.last_used = {}
        self.domain_preferences = {}

    def get_fingerprint(self, url: str) -> str:
        """Select appropriate browser fingerprint for URL"""
        domain = self._extract_domain(url)

        # Check if domain has preference
        if domain in self.domain_preferences:
            browser_type = self.domain_preferences[domain]
        else:
            # Weighted random selection
            browser_type = self._weighted_choice(self.BROWSER_WEIGHTS)
            self.domain_preferences[domain] = browser_type

        # Get specific version
        versions = self.BROWSER_VERSIONS[browser_type]

        # Avoid using same version twice in a row for domain
        last_version = self.last_used.get(domain)
        available_versions = [v for v in versions if v != last_version]

        if not available_versions:
            available_versions = versions

        selected = random.choice(available_versions)
        self.last_used[domain] = selected

        return selected

    def _extract_domain(self, url: str) -> str:
        from urllib.parse import urlparse
        return urlparse(url).netloc

    def _weighted_choice(self, weights: Dict[str, float]) -> str:
        choices = list(weights.keys())
        probabilities = list(weights.values())
        return random.choices(choices, weights=probabilities)[0]

class TLSClient:
    """Advanced TLS client with fingerprint bypass"""

    def __init__(self):
        self.rotator = FingerprintRotator()
        self.session_cache = {}
        self.ja3_stats = {}

    async def request(self, req: TLSRequest) -> TLSResponse:
        """Execute request with TLS fingerprint bypass"""

        # Select browser to impersonate
        if not req.impersonate:
            req.impersonate = self.rotator.get_fingerprint(req.url)

        # Configure session
        session_key = f"{req.url}:{req.impersonate}"

        if session_key in self.session_cache:
            session = self.session_cache[session_key]
        else:
            session = requests.Session()
            self.session_cache[session_key] = session

        # Build request parameters
        request_params = {
            "url": req.url,
            "method": req.method,
            "impersonate": req.impersonate,
            "timeout": req.timeout,
            "allow_redirects": req.follow_redirects
        }

        if req.headers:
            # Ensure headers match browser being impersonated
            request_params["headers"] = self._normalize_headers(
                req.headers,
                req.impersonate
            )

        if req.data:
            request_params["data"] = req.data
        elif req.json_data:
            request_params["json"] = req.json_data

        if req.proxy:
            request_params["proxies"] = {
                "http": req.proxy,
                "https": req.proxy
            }

        if req.cookies:
            request_params["cookies"] = req.cookies

        try:
            # Execute request with curl_cffi
            response = session.request(**request_params)

            # Extract JA3 fingerprint (for debugging)
            ja3 = self._get_ja3_fingerprint(req.impersonate)

            # Track JA3 usage
            self._track_ja3_usage(ja3, response.status_code)

            return TLSResponse(
                status_code=response.status_code,
                headers=dict(response.headers),
                content=response.text,
                cookies=dict(response.cookies),
                url=str(response.url),
                ja3_fingerprint=ja3,
                impersonation_used=req.impersonate
            )

        except Exception as e:
            raise HTTPException(
                status_code=500,
                detail=f"TLS request failed: {str(e)}"
            )

    def _normalize_headers(self, headers: Dict[str, str], impersonate: str) -> Dict[str, str]:
        """Normalize headers to match browser being impersonated"""

        browser_headers = {
            "chrome": {
                "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8",
                "Accept-Language": "en-US,en;q=0.9",
                "Accept-Encoding": "gzip, deflate, br, zstd",
                "Sec-Ch-Ua": '"Chromium";v="131", "Not_A Brand";v="24", "Google Chrome";v="131"',
                "Sec-Ch-Ua-Mobile": "?0",
                "Sec-Ch-Ua-Platform": '"Windows"',
                "Sec-Fetch-Dest": "document",
                "Sec-Fetch-Mode": "navigate",
                "Sec-Fetch-Site": "none",
                "Sec-Fetch-User": "?1",
                "Upgrade-Insecure-Requests": "1"
            },
            "firefox": {
                "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
                "Accept-Language": "en-US,en;q=0.5",
                "Accept-Encoding": "gzip, deflate, br",
                "Upgrade-Insecure-Requests": "1",
                "Sec-Fetch-Dest": "document",
                "Sec-Fetch-Mode": "navigate",
                "Sec-Fetch-Site": "none",
                "Sec-Fetch-User": "?1"
            },
            "safari": {
                "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
                "Accept-Language": "en-US,en;q=0.9",
                "Accept-Encoding": "gzip, deflate, br"
            }
        }

        # Determine browser type
        browser_type = "chrome"
        if "firefox" in impersonate.lower():
            browser_type = "firefox"
        elif "safari" in impersonate.lower():
            browser_type = "safari"

        # Merge with browser-specific headers
        normalized = browser_headers.get(browser_type, {}).copy()
        normalized.update(headers)

        return normalized

    def _get_ja3_fingerprint(self, impersonate: str) -> str:
        """Get JA3 fingerprint for impersonation profile"""
        # These are example JA3 hashes for common browsers
        ja3_database = {
            "chrome131": "772,4865-4866-4867-49195-49199-49196-49200-52393-52392-49171-49172-156-157-47-53,0-23-65281-10-11-35-16-5-13-18-51-45-43-27-17513-21,29-23-24,0",
            "chrome130": "772,4865-4866-4867-49195-49199-49196-49200-52393-52392-49171-49172-156-157-47-53,0-23-65281-10-11-35-16-5-13-18-51-45-43-27-17513-21,29-23-24,0",
            "firefox128": "771,4865-4867-4866-49195-49199-52393-52392-49196-49200-49162-49161-49171-49172-156-157-47-53,0-23-65281-10-11-35-16-5-34-51-43-13-45-28-21,29-23-24-25-256-257,0",
            "safari18": "771,4865-4867-4866-49196-49200-52393-52392-49195-49199-49162-49161-49171-49172-157-156-53-47-49160-49170-10,0-23-65281-10-11-16-5-13-18-51-45-43-27-21,29-23-24-25,0"
        }

        return ja3_database.get(impersonate, "unknown")

    def _track_ja3_usage(self, ja3: str, status_code: int):
        """Track JA3 fingerprint performance"""
        if ja3 not in self.ja3_stats:
            self.ja3_stats[ja3] = {
                "total": 0,
                "success": 0,
                "blocked": 0
            }

        stats = self.ja3_stats[ja3]
        stats["total"] += 1

        if status_code < 400:
            stats["success"] += 1
        elif status_code in [403, 429]:
            stats["blocked"] += 1

# Initialize global client
tls_client = TLSClient()

@app.post("/request", response_model=TLSResponse)
async def make_request(request: TLSRequest):
    """Execute HTTP request with TLS fingerprint bypass"""
    return await tls_client.request(request)

@app.get("/fingerprints")
async def list_fingerprints():
    """List available browser fingerprints"""
    return {
        "browsers": FingerprintRotator.BROWSER_VERSIONS,
        "weights": FingerprintRotator.BROWSER_WEIGHTS
    }

@app.get("/stats")
async def get_stats():
    """Get JA3 fingerprint usage statistics"""
    return {
        "ja3_stats": tls_client.ja3_stats,
        "session_count": len(tls_client.session_cache),
        "domain_preferences": tls_client.rotator.domain_preferences
    }
```

#### 1.2 Node.js Integration Layer

**File**: `apps/api/src/services/tls-client/index.ts`

```typescript
import fetch from 'node-fetch';
import { logger } from '../../lib/logger';

export interface TLSClientOptions {
  url: string;
  method?: string;
  headers?: Record<string, string>;
  body?: string | Buffer;
  json?: any;
  impersonate?: string;
  proxy?: string;
  timeout?: number;
  followRedirects?: boolean;
  cookies?: Record<string, string>;
}

export class TLSClient {
  private serviceUrl: string;
  private cache: Map<string, { ja3: string; lastUsed: Date }> = new Map();

  constructor(serviceUrl: string = 'http://localhost:8001') {
    this.serviceUrl = serviceUrl;
  }

  async request(options: TLSClientOptions): Promise<{
    status: number;
    headers: Record<string, string>;
    body: string;
    ja3: string;
  }> {
    try {
      const requestBody = {
        url: options.url,
        method: options.method || 'GET',
        headers: options.headers,
        data: typeof options.body === 'string' ? options.body : undefined,
        json_data: options.json,
        impersonate: options.impersonate || this.selectImpersonation(options.url),
        proxy: options.proxy,
        timeout: options.timeout || 30,
        follow_redirects: options.followRedirects !== false,
        cookies: options.cookies
      };

      const response = await fetch(`${this.serviceUrl}/request`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(requestBody)
      });

      if (!response.ok) {
        throw new Error(`TLS service error: ${response.status}`);
      }

      const result = await response.json();

      // Cache JA3 fingerprint
      this.cacheFingerprint(options.url, result.ja3_fingerprint);

      return {
        status: result.status_code,
        headers: result.headers,
        body: result.content,
        ja3: result.ja3_fingerprint
      };
    } catch (error) {
      logger.error('TLS client request failed', { error, url: options.url });
      throw error;
    }
  }

  private selectImpersonation(url: string): string {
    // Intelligent impersonation selection based on target
    const domain = new URL(url).hostname;

    // Use specific browsers for known sites
    const sitePreferences: Record<string, string> = {
      'google.com': 'chrome131',
      'facebook.com': 'chrome130',
      'amazon.com': 'chrome131',
      'linkedin.com': 'edge131',
      'twitter.com': 'firefox128',
      'instagram.com': 'safari18'
    };

    for (const [site, browser] of Object.entries(sitePreferences)) {
      if (domain.includes(site)) {
        return browser;
      }
    }

    // Random selection for unknown sites
    const browsers = ['chrome131', 'chrome130', 'firefox128', 'safari18', 'edge131'];
    return browsers[Math.floor(Math.random() * browsers.length)];
  }

  private cacheFingerprint(url: string, ja3: string): void {
    const domain = new URL(url).hostname;
    this.cache.set(domain, {
      ja3,
      lastUsed: new Date()
    });

    // Clean old entries
    if (this.cache.size > 1000) {
      const entries = Array.from(this.cache.entries());
      entries.sort((a, b) => a[1].lastUsed.getTime() - b[1].lastUsed.getTime());

      for (let i = 0; i < 100; i++) {
        this.cache.delete(entries[i][0]);
      }
    }
  }

  async getFingerprints(): Promise<any> {
    const response = await fetch(`${this.serviceUrl}/fingerprints`);
    return response.json();
  }

  async getStats(): Promise<any> {
    const response = await fetch(`${this.serviceUrl}/stats`);
    return response.json();
  }
}
```

### Phase 2: Enhanced Fetch Engine (Week 1-2)

#### 2.1 Replace Standard Fetch

**File**: `apps/api/src/scraper/scrapeURL/engines/tls-fetch/index.ts`

```typescript
import { EngineScrapeResult } from '..';
import { Meta } from '../..';
import { TLSClient } from '../../../services/tls-client';
import { SSLError } from '../../error';

const tlsClient = new TLSClient(process.env.TLS_SERVICE_URL || 'http://localhost:8001');

export async function scrapeURLWithTLSFetch(
  meta: Meta,
): Promise<EngineScrapeResult> {
  try {
    // Prepare request options
    const options = {
      url: meta.rewrittenUrl ?? meta.url,
      headers: {
        ...getDefaultHeaders(),
        ...meta.options.headers
      },
      timeout: meta.abort.scrapeTimeout(),
      proxy: meta.proxyUsed?.url
    };

    // Execute request with TLS fingerprint bypass
    const response = await tlsClient.request(options);

    // Log JA3 fingerprint for monitoring
    meta.logger.debug('TLS fetch completed', {
      url: meta.url,
      status: response.status,
      ja3: response.ja3
    });

    return {
      url: meta.url,
      html: response.body,
      statusCode: response.status,
      contentType: response.headers['content-type'],
      proxyUsed: meta.proxyUsed?.type || 'none',
      engine: 'tls-fetch',
      metadata: {
        ja3: response.ja3
      }
    };
  } catch (error) {
    if (error.message?.includes('SSL')) {
      throw new SSLError(meta.options.skipTlsVerification);
    }
    throw error;
  }
}

function getDefaultHeaders(): Record<string, string> {
  return {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
    'Accept-Language': 'en-US,en;q=0.9',
    'Accept-Encoding': 'gzip, deflate, br',
    'Cache-Control': 'no-cache',
    'Pragma': 'no-cache'
  };
}

export function tlsFetchMaxReasonableTime(meta: Meta): number {
  return 20000; // 20 seconds
}
```

### Phase 3: Advanced Features (Week 2)

#### 3.1 JA4+ Support

**File**: `apps/tls-service/ja4.py`

```python
import hashlib
from typing import Dict, List, Tuple

class JA4Generator:
    """Generate JA4+ fingerprints for advanced detection evasion"""

    def generate_ja4(self, client_hello: Dict) -> str:
        """Generate JA4 fingerprint from ClientHello"""

        # JA4 format: protocol,version,SNI,ciphersuites,extensions,signature_algos
        parts = []

        # Protocol (t=TLS, q=QUIC)
        parts.append('t' if client_hello.get('protocol') == 'TLS' else 'q')

        # TLS Version
        version = client_hello.get('version', '13')  # TLS 1.3
        parts.append(version)

        # SNI (d=domain, i=IP)
        parts.append('d' if client_hello.get('sni') else 'i')

        # Number of ciphersuites
        ciphers = client_hello.get('cipher_suites', [])
        parts.append(str(len(ciphers)).zfill(2))

        # Number of extensions
        extensions = client_hello.get('extensions', [])
        parts.append(str(len(extensions)).zfill(2))

        # First ALPN
        alpn = client_hello.get('alpn', ['h2'])[0][:2]
        parts.append(alpn)

        # Create hash
        ja4_raw = '_'.join(parts)

        # Add cipher suite hash
        cipher_hash = hashlib.sha256(
            ','.join(map(str, sorted(ciphers))).encode()
        ).hexdigest()[:12]

        # Add extension hash
        ext_hash = hashlib.sha256(
            ','.join(map(str, sorted(extensions))).encode()
        ).hexdigest()[:12]

        return f"{ja4_raw}_{cipher_hash}_{ext_hash}"

    def generate_ja4h(self, http_headers: Dict) -> str:
        """Generate JA4H (HTTP) fingerprint"""

        # Method and version
        method = http_headers.get('method', 'GET')[:2].lower()
        version = http_headers.get('version', '2')

        # Header count and order
        headers = http_headers.get('headers', {})
        header_order = list(headers.keys())
        header_count = str(len(headers)).zfill(3)

        # Accept headers
        accept_headers = [
            headers.get('Accept', ''),
            headers.get('Accept-Language', ''),
            headers.get('Accept-Encoding', '')
        ]

        # Create hash
        header_hash = hashlib.sha256(
            ','.join(header_order).encode()
        ).hexdigest()[:12]

        accept_hash = hashlib.sha256(
            ','.join(accept_headers).encode()
        ).hexdigest()[:12]

        return f"{method}{version}_{header_count}_{header_hash}_{accept_hash}"
```

## Testing & Validation

### Detection Testing Tools

1. **TLS Fingerprint Tests**
   - https://tls.browserleaks.com/
   - https://tlsfingerprint.io/
   - https://www.howsmyssl.com/

2. **JA3 Verification**
   ```bash
   # Capture and analyze TLS handshakes
   tcpdump -w capture.pcap -i any port 443
   tshark -r capture.pcap -Y "ssl.handshake.type==1" -T fields -e ssl.handshake.ciphersuite
   ```

3. **Performance Benchmarks**
   - Request success rate with/without TLS bypass
   - CAPTCHA encounter rate reduction
   - Detection rate by major WAFs

## Monitoring & Maintenance

### Metrics to Track
```typescript
interface TLSMetrics {
  fingerprintRotations: number;
  ja3Uniqueness: number;
  detectionRate: Map<string, number>;
  captchaRate: Map<string, number>;
  wafBypassRate: Map<string, number>;
}
```

### Update Schedule
- **Weekly**: Update browser versions in fingerprint database
- **Bi-weekly**: Test against major WAF updates
- **Monthly**: Analyze detection patterns and adjust strategies

## Docker Configuration

**File**: `apps/tls-service/Dockerfile`

```dockerfile
FROM python:3.12-slim

# Install curl-impersonate dependencies
RUN apt-get update && apt-get install -y \
    libcurl4-openssl-dev \
    libssl-dev \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Run service
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001"]
```

## Success Criteria

1. **Undetectable TLS fingerprints**: 0% detection rate on major testing sites
2. **Dynamic fingerprint rotation**: Different JA3 hash per request
3. **WAF bypass rate**: >85% success against Cloudflare, DataDome, PerimeterX
4. **Performance**: <100ms overhead per request
5. **Stability**: 99.9% uptime for TLS service

## Implementation Timeline

- **Week 1**: Python TLS service with curl_cffi
- **Week 2**: Integration with existing engines, testing
- **Week 3**: JA4+ support and advanced features
- **Week 4**: Production deployment and monitoring

## Dependencies

### Python (TLS Service)
```requirements.txt
curl-cffi[speedups]>=0.7.3
fastapi>=0.115.0
uvicorn>=0.32.0
pydantic>=2.9.0
redis>=5.2.0
httpx>=0.28.0
```

### Node.js (Integration)
```json
{
  "tls-client": "^1.0.0",
  "node-fetch": "^3.3.2"
}
```

## Cost Analysis

### Development
- 8 days development
- 2 days testing
- 2 days integration

### Infrastructure
- Python microservice: 256MB RAM, 0.5 CPU
- Minimal network overhead (<100ms per request)
- No additional licensing costs

### Expected ROI
- 80% reduction in Cloudflare challenges
- 75% reduction in DataDome blocks
- 90% improvement in scraping success for protected sites