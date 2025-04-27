# Optimizing-Headless-Browser-Traffic
When using Puppeteer for data scraping, traffic consumption is an important consideration. Especially when using proxy services, traffic costs can increase significantly. To optimize traffic usage, we can adopt the following strategies
## Overview
When using Puppeteer for data scraping, traffic consumption is an important consideration.  Especially when using proxy services, traffic costs can increase significantly. To optimize traffic usage, we can adopt the following strategies:
1. **Resource interception**: Reduce traffic consumption by intercepting unnecessary resource requests.
2. **Request URL interception**: Further reduce traffic by intercepting specific requests based on URL characteristics.
3. **Simulate mobile devices**: Use mobile device configurations to obtain lighter page versions.
4. **Comprehensive optimization**: Combine the above methods to achieve the best results.

## Optimization Scheme 1: Resource Interception

### Resource Interception Introduction

In Puppeteer, `page.setRequestInterception(true)` can capture every network request initiated by the browser and decide to **continue** (`request.continue()`), **terminate** (`request.abort()`), or **customize the response** (`request.respond()`).

This method can significantly reduce bandwidth consumption, especially suitable for **crawling**, **screenshotting**, and **performance optimization** scenarios.

### Interceptable Resource Types and Suggestions

| Resource Type          | Description            | Example                      | Impact After Interception             | Recommendation    |
|---------------|---------------|-------------------------|-------------------|-------|
| `image`       | Image resources          | JPG/PNG/GIF/WebP images    | Images will not be displayed         | ⭐ Safe  |
| `font`        | Font files          | TTF/WOFF/WOFF2 fonts      | System default fonts will be used instead        | ⭐ Safe  |
| `media`       | Media files          | Video/audio files                 | Media content cannot be played          | ⭐ Safe  |
| `manifest`    | Web App Manifest      | PWA configuration file                | PWA functionality may be affected       | ⭐ Safe  |
| `prefetch`    | Prefetch resources          | `<link rel="prefetch">` | Minimal impact on the page           | ⭐ Safe  |
| `stylesheet`  | CSS Stylesheet       | External CSS files               | Page styles are lost, may affect layout     | ⚠️ Caution |
| `websocket`   | WebSocket     | Real-time communication connection                  | Real-time functionality disabled            | ⚠️ Caution |
| `eventsource` | Server-Sent Events       | Server push data                 | Push functionality disabled            | ⚠️ Caution |
| `preflight`   | CORS preflight request     | OPTIONS request              | Cross-origin requests fail            | ⚠️ Caution |
| `script`      | JavaScript scripts | External JS files                | Dynamic functionality disabled, SPA may not render | ❌ Avoid  |
| `xhr`         | XHR requests        | AJAX data requests               | Unable to obtain dynamic data          | ❌ Avoid  |
| `fetch`       | Fetch requests      | Modern AJAX requests              | Unable to obtain dynamic data          | ❌ Avoid  |
| `document`    | Main document           | HTML page itself               | Page cannot load            | ❌ Avoid  |

**Recommendation Level Explanation:**

- ⭐ **Safe**: Interception has almost no impact on data scraping or first-screen rendering; it is recommended to block by default.
- ⚠️ **Caution**: May break styles, real-time functions, or cross-origin requests; requires business judgment.
- ❌ **Avoid**: High probability of causing SPA/dynamic sites to fail to render or obtain data normally, unless you are absolutely sure you don't need these resources.

### Resource Interception Example Code

```javascript
import puppeteer from 'puppeteer-core';

const scrapelessUrl = 'wss://browser.scrapeless.com/browser?token=your_api_key&session_ttl=180&proxy_country=ANY';

async function scrapeWithResourceBlocking(url) {
    const browser = await puppeteer.connect({
        browserWSEndpoint: scrapelessUrl,
        defaultViewport: null
    });
    const page = await browser.newPage();

    // Enable request interception
    await page.setRequestInterception(true);

    // Define resource types to block
    const BLOCKED_TYPES = new Set([
        'image',
        'font',
        'media',
        'stylesheet',
    ]);

    // Intercept requests
    page.on('request', (request) => {
        if (BLOCKED_TYPES.has(request.resourceType())) {
            request.abort();
            console.log(`Blocked: ${request.resourceType()} - ${request.url().substring(0, 50)}...`);
        } else {
            request.continue();
        }
    });

    await page.goto(url, {waitUntil: 'domcontentloaded'});

    // Extract data
    const data = await page.evaluate(() => {
        return {
            title: document.title,
            content: document.body.innerText.substring(0, 1000)
        };
    });

    await browser.close();
    return data;
}

// Usage
scrapeWithResourceBlocking('https://www.scrapeless.com')
    .then(data => console.log('Scraping result:', data))
    .catch(error => console.error('Scraping failed:', error));
```

## Optimization Scheme 2: Request URL Interception

In addition to intercepting by resource type, more granular interception control can be performed based on URL characteristics. This is particularly effective for blocking ads, analytics scripts, and other unnecessary third-party requests.

### URL Interception Strategies

1. **Intercept by domain**: Block all requests from a specific domain
2. **Intercept by path**: Block requests from a specific path
3. **Intercept by file type**: Block files with specific extensions
4. **Intercept by keyword**: Block requests whose URLs contain specific keywords

### Common Interceptable URL Patterns

| URL Pattern  | Description        | Example                                             | Recommendation    |
|--------|-----------|------------------------------------------------|-------|
| Advertising services   | Advertising network domains    | `ad.doubleclick.net`, `googleadservices.com`   | ⭐ Safe  |
| Analytics services   | Statistics and analytics scripts   | `google-analytics.com`, `hotjar.com`           | ⭐ Safe  |
| Social media plugins | Social sharing buttons, etc.   | `platform.twitter.com`, `connect.facebook.net` | ⭐ Safe  |
| Tracking pixels   | Pixels that track user behavior | URLs containing `pixel`, `beacon`, `tracker`              | ⭐ Safe  |
| Large media files | Large video, audio files | Extensions like `.mp4`, `.webm`, `.mp3`                    | ⭐ Safe  |
| Font services   | Online font services    | `fonts.googleapis.com`, `use.typekit.net`      | ⭐ Safe  |
| CDN resources  | Static resource CDN   | `cdn.jsdelivr.net`, `unpkg.com`                | ⚠️ Caution |

### URL Interception Example Code

```javascript
import puppeteer from 'puppeteer-core';

const scrapelessUrl = 'wss://browser.scrapeless.com/browser?token=your_api_key&session_ttl=180&proxy_country=ANY';

async function scrapeWithUrlBlocking(url) {
    const browser = await puppeteer.connect({
        browserWSEndpoint: scrapelessUrl,
        defaultViewport: null
    });
    const page = await browser.newPage();

    // Enable request interception
    await page.setRequestInterception(true);

    // Define domains and URL patterns to block
    const BLOCKED_DOMAINS = [
        'google-analytics.com',
        'googletagmanager.com',
        'doubleclick.net',
        'facebook.net',
        'twitter.com',
        'linkedin.com',
        'adservice.google.com',
    ];

    const BLOCKED_PATHS = [
        '/ads/',
        '/analytics/',
        '/pixel/',
        '/tracking/',
        '/stats/',
    ];

    // Intercept requests
    page.on('request', (request) => {
        const url = request.url();

        // Check domain
        if (BLOCKED_DOMAINS.some(domain => url.includes(domain))) {
            request.abort();
            console.log(`Blocked domain: ${url.substring(0, 50)}...`);
            return;
        }

        // Check path
        if (BLOCKED_PATHS.some(path => url.includes(path))) {
            request.abort();
            console.log(`Blocked path: ${url.substring(0, 50)}...`);
            return;
        }

        // Allow other requests
        request.continue();
    });

    await page.goto(url, {waitUntil: 'domcontentloaded'});

    // Extract data
    const data = await page.evaluate(() => {
        return {
            title: document.title,
            content: document.body.innerText.substring(0, 1000)
        };
    });

    await browser.close();
    return data;
}

// Usage
scrapeWithUrlBlocking('https://www.scrapeless.com')
    .then(data => console.log('Scraping result:', data))
    .catch(error => console.error('Scraping failed:', error));
```

## Optimization Scheme 3: Simulate Mobile Devices

Simulating mobile devices is another effective traffic optimization strategy because mobile websites usually provide lighter page content.

### Advantages of Mobile Device Simulation

1. **Lighter page versions**: Many websites provide more concise content for mobile devices
2. **Smaller image resources**: Mobile versions usually load smaller images
3. **Simplified CSS and JavaScript**: Mobile versions usually use simplified styles and scripts
4. **Reduced ads and non-core content**: Mobile versions often remove some non-core functionality
5. **Adaptive response**: Obtain content layouts optimized for small screens

### Mobile Device Simulation Configuration

Here are the configuration parameters for several commonly used mobile devices:

```javascript
const iPhoneX = {
    viewport: {
        width: 375,
        height: 812,
        deviceScaleFactor: 3,
        isMobile: true,
        hasTouch: true,
        isLandscape: false
    }
};
```

Or directly use the built-in methods of puppeteer to simulate mobile devices

``` javascript
import { KnownDevices } from 'puppeteer-core';
const iPhone = KnownDevices['iPhone 15 Pro'];

const browser = await puppeteer.launch();
const page = await browser.newPage();
await page.emulate(iPhone);
```

### Mobile Device Simulation Example Code

```javascript
import puppeteer, {KnownDevices} from 'puppeteer-core';

const scrapelessUrl = 'wss://browser.scrapeless.com/browser?token=your_api_key&session_ttl=180&proxy_country=ANY';

async function scrapeWithMobileEmulation(url) {
    const browser = await puppeteer.connect({
        browserWSEndpoint: scrapelessUrl,
        defaultViewport: null
    });

    const page = await browser.newPage();

    // Set mobile device simulation
    const iPhone = KnownDevices['iPhone 15 Pro'];
    await page.emulate(iPhone);

    await page.goto(url, {waitUntil: 'domcontentloaded'});
    // Extract data
    const data = await page.evaluate(() => {
        return {
            title: document.title,
            content: document.body.innerText.substring(0, 1000)
        };
    });

    await browser.close();
    return data;
}

// Usage
scrapeWithMobileEmulation('https://www.scrapeless.com')
    .then(data => console.log('Scraping result:', data))
    .catch(error => console.error('Scraping failed:', error));
```

## Comprehensive Optimization Example

Here is a comprehensive example combining all optimization schemes:

```javascript
import puppeteer, {KnownDevices} from 'puppeteer-core';

const scrapelessUrl = 'wss://browser.scrapeless.com/browser?token=your_api_key&session_ttl=180&proxy_country=ANY';

async function optimizedScraping(url) {
    console.log(`Starting optimized scraping: ${url}`);

    // Record traffic usage
    let totalBytesUsed = 0;

    const browser = await puppeteer.connect({
        browserWSEndpoint: scrapelessUrl,
        defaultViewport: null
    });

    const page = await browser.newPage();

    // Set mobile device simulation
    const iPhone = KnownDevices['iPhone 15 Pro'];
    await page.emulate(iPhone);

    // Set request interception
    await page.setRequestInterception(true);

    // Define resource types to block
    const BLOCKED_TYPES = [
        'image',
        'media',
        'font'
    ];

    // Define domains to block
    const BLOCKED_DOMAINS = [
        'google-analytics.com',
        'googletagmanager.com',
        'facebook.net',
        'doubleclick.net',
        'adservice.google.com'
    ];

    // Define URL paths to block
    const BLOCKED_PATHS = [
        '/ads/',
        '/analytics/',
        '/tracking/'
    ];

    // Intercept requests
    page.on('request', (request) => {
        const url = request.url();
        const resourceType = request.resourceType();

        // Check resource type
        if (BLOCKED_TYPES.includes(resourceType)) {
            console.log(`Blocked resource type: ${resourceType} - ${url.substring(0, 50)}...`);
            request.abort();
            return;
        }

        // Check domain
        if (BLOCKED_DOMAINS.some(domain => url.includes(domain))) {
            console.log(`Blocked domain: ${url.substring(0, 50)}...`);
            request.abort();
            return;
        }

        // Check path
        if (BLOCKED_PATHS.some(path => url.includes(path))) {
            console.log(`Blocked path: ${url.substring(0, 50)}...`);
            request.abort();
            return;
        }

        // Allow other requests
        request.continue();
    });

    // Monitor network traffic
    page.on('response', async (response) => {
        const headers = response.headers();
        const contentLength = headers['content-length'] ? parseInt(headers['content-length'], 10) : 0;
        totalBytesUsed += contentLength;
    });

    await page.goto(url, {waitUntil: 'domcontentloaded'});

    // Simulate scrolling to trigger lazy-loading content
    await page.evaluate(() => {
        window.scrollBy(0, window.innerHeight);
    });

    await new Promise(resolve => setTimeout(resolve, 1000))

    // Extract data
    const data = await page.evaluate(() => {
        return {
            title: document.title,
            content: document.body.innerText.substring(0, 1000),
            links: Array.from(document.querySelectorAll('a')).slice(0, 10).map(a => ({
                text: a.innerText,
                href: a.href
            }))
        };
    });

    // Output traffic usage statistics
    console.log(`\nTraffic Usage Statistics:`);
    console.log(`Used: ${(totalBytesUsed / 1024 / 1024).toFixed(2)} MB`);

    await browser.close();
    return data;
}

// Usage
optimizedScraping('https://www.scrapeless.com')
    .then(data => console.log('Scraping complete:', data))
    .catch(error => console.error('Scraping failed:', error));
```

### Optimization Comparison

We try removing the optimized code from the comprehensive example to compare the traffic before and after optimization. Here is the unoptimized example code:

``` javascript
import puppeteer from 'puppeteer-core';

const scrapelessUrl = 'wss://browser.scrapeless.com/browser?token=your_api_key&session_ttl=180&proxy_country=ANY';

async function optimizedScraping(url) {
  console.log(`Starting optimized scraping: ${url}`);

  // Record traffic usage
  let totalBytesUsed = 0;

  const browser = await puppeteer.connect({
    browserWSEndpoint: scrapelessUrl,
    defaultViewport: null
  });

  const page = await browser.newPage();

  // Set request interception
  await page.setRequestInterception(true);

  // Intercept requests
  page.on('request', (request) => {
    request.continue();
  });

  // Monitor network traffic
  page.on('response', async (response) => {
    const headers = response.headers();
    const contentLength = headers['content-length'] ? parseInt(headers['content-length'], 10) : 0;
    totalBytesUsed += contentLength;
  });

  await page.goto(url, {waitUntil: 'domcontentloaded'});

  // Simulate scrolling to trigger lazy-loading content
  await page.evaluate(() => {
    window.scrollBy(0, window.innerHeight);
  });

  await new Promise(resolve => setTimeout(resolve, 1000))

  // Extract data
  const data = await page.evaluate(() => {
    return {
      title: document.title,
      content: document.body.innerText.substring(0, 1000),
      links: Array.from(document.querySelectorAll('a')).slice(0, 10).map(a => ({
        text: a.innerText,
        href: a.href
      }))
    };
  });

  // Output traffic usage statistics
  console.log(`\nTraffic Usage Statistics:`);
  console.log(`Used: ${(totalBytesUsed / 1024 / 1024).toFixed(2)} MB`);

  await browser.close();
  return data;
}

// Usage
optimizedScraping('https://www.scrapeless.com')
  .then(data => console.log('Scraping complete:', data))
  .catch(error => console.error('Scraping failed:', error));
```

After running the unoptimized code, we can see the traffic difference very intuitively from the printed information:

| Scenario      | Traffic Used (MB) |     Saving Ratio     |
|---------|:---------------:|:------------:|
| Unoptimized     |      6.03       |      —       |
| **Optimized** |    **0.81**     | **≈ 86.6 %** |

By combining the above optimization schemes, proxy traffic consumption can be significantly reduced, scraping efficiency can be improved, and ensuring that the required core content is obtained.
