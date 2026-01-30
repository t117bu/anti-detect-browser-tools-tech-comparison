# Botasaurus - Technical Analysis

> **Repository:** [omkarcloud/botasaurus](https://github.com/omkarcloud/botasaurus)
> **Category:** Browser Automation / Anti-Detection Framework
> **Language:** Python
> **Last Analyzed:** January 2026

## Overview

Botasaurus is a Python-based web scraping framework that wraps Selenium with anti-detection capabilities. It markets itself as an "undefeatable" scraping solution that can bypass major bot detection systems.

## Verdict

| Aspect | Rating | Notes |
|--------|--------|-------|
| **Overall Quality** | ⭐⭐⭐⭐ | Well-engineered, good for mid-tier protection |
| **Ease of Use** | ⭐⭐⭐⭐⭐ | Excellent decorator-based API |
| **Anti-Detection** | ⭐⭐⭐ | Good basics, not "undefeatable" |
| **Documentation** | ⭐⭐⭐⭐ | Comprehensive with examples |
| **Active Development** | ⭐⭐⭐⭐ | Regular updates |

**TL;DR:** Solid framework for evading basic-to-medium bot detection. Won't defeat enterprise-grade systems alone.

---

## How It Works

### It's NOT Just JavaScript Injection

Botasaurus uses a multi-layered approach combining several techniques:

### 1. Chrome DevTools Protocol (CDP) Mouse Events

Instead of Selenium's standard `.click()`, it dispatches raw mouse events via CDP:

```python
driver.run_cdp_command(
    cdp.input_.dispatch_mouse_event(
        "mousePressed",
        x=x, y=y,
        modifiers=0, buttons=1,
        button=cdp.input_.MouseButton("left"),
        click_count=1,
    )
)
```

**Why it works:** Standard Selenium clicks have machine-perfect coordinates and timing. CDP events mimic how a real browser generates events internally.

### 2. Bézier Curve Mouse Movement

The `HumanizeMouseTrajectory` class generates mathematically realistic mouse paths:

- **Bernstein polynomials** for smooth Bézier curves
- **Gaussian noise** applied to ~50% of intermediate points
- **Easing functions** (easeOutQuad, easeInOutSine) for natural acceleration
- **100+ interpolation points** per movement

```python
class HumanizeMouseTrajectory:
    def generate_curve(self, **kwargs):
        # Generate internal knots for curve control points
        internalKnots = self.generate_internal_knots(...)
        # Calculate Bézier curve points
        points = self.generate_points(internalKnots)
        # Add human-like distortion
        points = self.distort_points(points, distortion_mean, distortion_st_dev, distortion_frequency)
        # Apply easing for natural speed variation
        points = self.tween_points(points, tween, target_points)
        return points
```

**Why it works:** Humans don't move mice in straight lines. They overshoot, have micro-tremors, and accelerate/decelerate naturally. Detection systems flag perfect linear movements.

### 3. JavaScript Property Patching

Fixes CDP mouse event fingerprint leakage:

```javascript
Object.defineProperty(MouseEvent.prototype, 'screenX', {
    get: function () {
        return this.clientX + window.screenX;
    }
});
Object.defineProperty(MouseEvent.prototype, 'screenY', {
    get: function () {
        return this.clientY + window.screenY;
    }
});
```

**Why it works:** CDP-dispatched events have incorrect `screenX`/`screenY` values. This patches MouseEvent.prototype before detection scripts can check it.

### 4. Referrer Spoofing

- `google_get()` - Navigates to Google first, then to target
- `get_by_current_page_referrer()` - Sets proper HTTP referrer headers

**Why it works:** Direct visits with no referrer are suspicious. Real users click links from search engines or other sites.

### 5. Timing Randomization

```python
def random_natural_sleep(self):
    sleep(random.randint(170, 280) / 1000)
```

Random 170-280ms delays between actions.

**Why it works:** Bots act instantly or with fixed delays. Humans have variable reaction times.

---

## Anti-Detection Effectiveness

| Technique | What It Evades | Effectiveness |
|-----------|---------------|---------------|
| CDP Mouse Events | `event.isTrusted` checks, coordinate analysis | ⭐⭐⭐⭐⭐ |
| Bézier Curves | Mouse trajectory analysis | ⭐⭐⭐⭐⭐ |
| screenX/Y patching | CDP fingerprint leakage | ⭐⭐⭐ |
| Referrer spoofing | Direct-visit detection | ⭐⭐ |
| Random timing | Timing pattern analysis | ⭐⭐⭐ |

---

## What It Claims to Bypass

According to their documentation:

- ✅ Cloudflare Web Application Firewall (WAF)
- ✅ BrowserScan Bot Detection
- ✅ Fingerprint.com Bot Detection
- ✅ DataDome Bot Detection
- ✅ Cloudflare Turnstile CAPTCHA

**Reality check:** These bypasses work under specific conditions and may not be reliable long-term.

---

## Limitations

### What It Does NOT Handle

1. **Advanced Fingerprinting**
   - Canvas fingerprinting
   - WebGL fingerprinting
   - Audio context fingerprinting
   - Font enumeration
   - Hardware concurrency checks

2. **TLS Fingerprinting**
   - Browser TLS handshake signatures are unique and hard to spoof

3. **Behavioral Analysis at Scale**
   - Cloudflare tracks patterns across millions of requests
   - Unusual browsing patterns trigger detection regardless of mouse movements

4. **External Dependency**
   - Core detection evasion relies on `botasaurus_driver` package
   - `navigator.webdriver` patching handled externally

5. **IP Reputation**
   - Datacenter IPs will still be flagged
   - Requires residential proxies for serious scraping

---

## Architecture

```
botasaurus/
├── botasaurus/              # Main framework (decorators, config)
│   ├── browser_decorator.py # @browser decorator
│   ├── request_decorator.py # @request decorator
│   └── cache.py             # Result caching
├── botasaurus_humancursor/  # Human mouse simulation
│   ├── human_curve_generator.py  # Bézier curves
│   ├── web_cursor.py        # Click/drag operations
│   └── web_adjuster.py      # JS patching + movement
├── botasaurus_driver/       # (External) Stealth WebDriver
├── botasaurus_requests/     # HTTP request handling
└── botasaurus_api/          # REST API wrapper
```

---

## Code Quality

**Pros:**
- Clean decorator-based API
- Well-structured modules
- Good separation of concerns
- Comprehensive helper methods

**Cons:**
- Some commented-out debug code
- Relies heavily on external packages
- Limited error handling in some areas

---

## Use Cases

### Good For
- Scraping sites with basic bot detection
- E-commerce data collection
- News/content aggregation
- Academic research

### Not Ideal For
- High-security targets (banking, enterprise systems)
- Large-scale scraping without proxies
- Sites using Akamai Bot Manager, PerimeterX Enterprise
- Real-time scraping at high frequency

---

## Comparison to Alternatives

| Feature | Botasaurus | Puppeteer-Stealth | Playwright | undetected-chromedriver |
|---------|------------|-------------------|------------|------------------------|
| Language | Python | JavaScript | Multi | Python |
| Human Mouse | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐ |
| API Simplicity | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Fingerprint Evasion | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| Active Development | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

---

## Quick Start

```python
from botasaurus.browser import browser, Driver

@browser(
    headless=True,
    block_images_and_css=True,
)
def scrape_task(driver: Driver, data):
    driver.google_get("https://example.com")
    driver.short_random_sleep()
    return driver.text("h1")

result = scrape_task()
```

---

## Bottom Line

Botasaurus is a **legitimate improvement** over raw Selenium. The human-like mouse movements via Bézier curves and CDP integration are well-implemented and will help evade basic detection.

However, calling it "undefeatable" is marketing hyperbole. For serious anti-bot systems, you'll still need:

- Residential/mobile proxies
- Real browser profiles with history
- More sophisticated fingerprint management
- Rate limiting and behavioral variation
- Potentially CAPTCHA solving services

**Recommendation:** Good choice for mid-tier scraping needs. Pair with quality proxies and realistic browsing patterns for best results.

---

## Links

- [GitHub Repository](https://github.com/omkarcloud/botasaurus)
- [Documentation](https://github.com/omkarcloud/botasaurus#readme)
- [Bot Detection Test Code](https://github.com/omkarcloud/botasaurus/blob/master/bot_detection_tests.py)
