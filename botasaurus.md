# Botasaurus - Technical Analysis

> **Repository:** [omkarcloud/botasaurus](https://github.com/omkarcloud/botasaurus)
> **Category:** Browser Automation / Anti-Detection Framework
> **Language:** Python
> **Type:** Selenium wrapper with human simulation

---

## Overview

Botasaurus is a Python-based web scraping framework that wraps Selenium with anti-detection capabilities. It markets itself as an "undefeatable" scraping solution that can bypass major bot detection systems.

## Verdict

| Aspect | Rating | Notes |
|--------|--------|-------|
| **Overall Quality** | ⭐⭐⭐⭐ | Well-engineered, good for mid-tier protection |
| **Ease of Use** | ⭐⭐⭐⭐⭐ | Excellent decorator-based API |
| **Anti-Detection** | ⭐⭐⭐ | Good basics, not "undefeatable" |
| **Human Simulation** | ⭐⭐⭐⭐⭐ | Best-in-class Bézier curve mouse movements |
| **Documentation** | ⭐⭐⭐⭐ | Comprehensive with examples |

**TL;DR:** Solid framework for evading basic-to-medium bot detection with excellent human-like behavior simulation. Won't defeat enterprise-grade systems alone.

---

## How It Works - Technical Deep Dive

### It's NOT Just JavaScript Injection

Botasaurus uses a **multi-layered approach** combining several techniques at different levels:

### 1. Chrome DevTools Protocol (CDP) Mouse Events

Instead of Selenium's standard `.click()` which has detectable patterns, it dispatches raw mouse events via CDP:

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

**Why it works:** Standard Selenium clicks use machine-perfect coordinates and timing. CDP events mimic how a real browser generates events internally, making them indistinguishable from real user interactions.

### 2. Bézier Curve Mouse Movement (Best-in-Class)

The `HumanizeMouseTrajectory` class in `botasaurus_humancursor/human_curve_generator.py` generates mathematically realistic mouse paths:

```python
class HumanizeMouseTrajectory:
    def generate_curve(self, **kwargs):
        # Generate internal knots for curve control points
        internalKnots = self.generate_internal_knots(...)

        # Calculate Bézier curve points using Bernstein polynomials
        points = self.generate_points(internalKnots)

        # Add human-like distortion (Gaussian noise)
        points = self.distort_points(
            points,
            distortion_mean=1,
            distortion_st_dev=1,
            distortion_frequency=0.5  # ~50% of points get noise
        )

        # Apply easing for natural speed variation
        points = self.tween_points(points, tween, target_points=100)
        return points
```

**Technical details:**
- **Bernstein polynomials** for smooth Bézier curves
- **Gaussian noise** applied to ~50% of intermediate points (mean=1, std_dev=1)
- **Easing functions** via pytweening (easeOutQuad, easeInOutSine)
- **100+ interpolation points** per movement for granular trajectory
- **Random knots** (0-3) create varied curve shapes

**Why it works:** Humans don't move mice in straight lines. They overshoot, have micro-tremors, and accelerate/decelerate naturally. Detection systems flag perfect linear movements. This implementation is the most sophisticated among the tools analyzed.

### 3. JavaScript Property Patching

Fixes CDP mouse event fingerprint leakage in `web_adjuster.py`:

```javascript
// CDP-dispatched events have incorrect screenX/screenY values
// This patches MouseEvent.prototype BEFORE detection scripts run

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

**Why it works:** CDP-dispatched events leak their synthetic origin through incorrect `screenX`/`screenY` values. This patches the prototype before detection scripts can check these properties.

### 4. Referrer Spoofing

```python
# Simulates clicking a Google search result
driver.google_get("https://target-site.com")
# Flow: Navigate to Google → "search" → click result → arrive at target

# Uses current page as referrer for subsequent navigation
driver.get_by_current_page_referrer("https://target-site.com")
```

**Why it works:** Direct visits with no referrer are suspicious. Real users typically arrive via search engines or links from other sites. This simulates natural navigation patterns.

### 5. Timing Randomization

```python
def random_natural_sleep(self):
    # Random 170-280ms delays (human reaction time range)
    sleep(random.randint(170, 280) / 1000)
```

**Why it works:** Bots act instantly or with fixed delays. Humans have variable reaction times typically in the 100-500ms range.

### 6. Browser Profile Persistence

```python
@browser(
    profile="my_profile",        # Full Chrome profile with history
    tiny_profile=True,           # Lightweight 1KB profile alternative
    user_agent=UserAgent.HASHED, # Consistent UA per profile
)
```

- **Full profiles**: Chrome profiles with cookies, history, localStorage
- **Tiny profiles** (1KB each): Lightweight session persistence without full profile overhead

**Why it works:** Websites track user history and behavioral patterns. Profiles with browsing history appear more legitimate than fresh instances.

---

## Architecture

```
botasaurus/
├── botasaurus/                         # Main framework
│   ├── browser_decorator.py            # @browser decorator
│   ├── request_decorator.py            # @request decorator
│   ├── browser.py                      # Driver class
│   └── cache.py                        # Result caching
├── botasaurus_humancursor/             # Human simulation (KEY)
│   ├── human_curve_generator.py        # Bézier curves
│   ├── web_cursor.py                   # Click/drag operations
│   ├── web_adjuster.py                 # JS patching
│   └── calculate_and_randomize.py      # Curve parameters
├── botasaurus_driver/ (external)       # Stealth WebDriver
├── botasaurus_api/                     # REST API wrapper
└── botasaurus_server/                  # UI/Server
```

---

## Anti-Detection Effectiveness

| Technique | What It Evades | Effectiveness |
|-----------|---------------|---------------|
| CDP Mouse Events | `event.isTrusted` checks, coordinate analysis | ⭐⭐⭐⭐⭐ |
| Bézier Curves | Mouse trajectory analysis | ⭐⭐⭐⭐⭐ |
| screenX/Y patching | CDP fingerprint leakage | ⭐⭐⭐ |
| Referrer spoofing | Direct-visit detection | ⭐⭐ |
| Random timing | Timing pattern analysis | ⭐⭐⭐ |
| Profile persistence | Session/history analysis | ⭐⭐⭐ |

---

## Services Bypassed

According to documentation and `bot_detection_tests.py`:

| Status | Service |
|--------|---------|
| ✅ Bypassed | Cloudflare WAF, Cloudflare Turnstile |
| ✅ Bypassed | BrowserScan Bot Detection |
| ✅ Bypassed | Fingerprint.com Bot Detection |
| ✅ Bypassed | DataDome Bot Detection |
| ❌ Not Effective | Akamai Bot Manager (enterprise) |
| ❌ Not Effective | PerimeterX Enterprise |
| ❌ Not Effective | Canvas/WebGL fingerprinting |
| ❌ Not Effective | TLS fingerprinting |

**Reality check:** These bypasses work under specific conditions and may not be reliable long-term.

---

## What It Does NOT Handle

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
   - Unusual patterns trigger detection regardless of mouse movements

4. **External Dependency**
   - Core detection evasion relies on `botasaurus_driver` package
   - `navigator.webdriver` patching handled externally

5. **IP Reputation**
   - Datacenter IPs will still be flagged
   - Requires residential proxies for serious scraping

---

## Usage Example

```python
from botasaurus.browser import browser, Driver

@browser(
    headless=False,              # Headed mode recommended
    block_images_and_css=True,   # Reduce detection surface
    proxy="http://user:pass@host:port",
    profile="my_profile",
    user_agent=UserAgent.HASHED,
)
def scrape_site(driver: Driver, data):
    # Human-like navigation with referrer
    driver.google_get("https://example.com")

    # Enable human mouse movements
    driver.enable_human_mode()

    # Click with Bézier curves
    driver.click(".login-button")

    # Type with random delays
    driver.type(".username", "user@example.com")

    # Natural sleep
    driver.short_random_sleep()

    # Get parsed HTML
    return driver.bs4.select(".results")

# Run with parallelization
results = scrape_site(data=["url1", "url2", "url3"])
```

---

## Pros and Cons

### Advantages

| Advantage | Details |
|-----------|---------|
| **Human Mouse Movement** | ⭐⭐⭐⭐⭐ Best-in-class Bézier curve implementation |
| **Ease of Use** | Decorator-based API is extremely intuitive |
| **All-in-One** | Scraping + UI + Desktop apps + REST API |
| **Parallelization** | Easy concurrent browser management |
| **Caching** | Built-in result caching with expiration |
| **Profile Management** | Lightweight tiny profiles (1KB each) |
| **Active Development** | Regular updates (v4.0.97) |
| **MIT License** | Open source |

### Limitations

| Limitation | Details |
|------------|---------|
| **"Undefeatable" Claim** | Marketing hyperbole - not truly undefeatable |
| **Headless Mode** | Won't work for high-security targets |
| **Advanced Fingerprinting** | Doesn't handle canvas/WebGL/audio fingerprinting |
| **TLS Fingerprinting** | Can't spoof browser TLS handshake signatures |
| **External Dependency** | Relies on botasaurus_driver for core evasion |
| **Python Only** | No JavaScript/Node.js version |
| **Resource Intensive** | Chrome browser requires significant memory |

---

## Comparison with Alternatives

| Feature | Botasaurus | Patchright | SeleniumBase | BotBrowser |
|---------|:----------:|:----------:|:------------:|:----------:|
| **Human Mouse** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **CDP Evasion** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Fingerprint Control** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Ease of Use** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Language** | Python | Multi | Python | Multi |
| **Cost** | Free | Free | Free | $$ |

---

## When to Use Botasaurus

### Best For:
- Web scraping applications (not just automation)
- Projects needing **human-like mouse movements**
- Building scraper UIs for non-technical users
- Mid-tier protected sites (Cloudflare, DataDome basic)
- Python-only environments

### Not Ideal For:
- Enterprise-grade anti-bot systems (Akamai, PerimeterX)
- High-speed automation (millions of requests)
- When you need headless mode for protected sites
- Sites with advanced fingerprinting

---

## Bottom Line

Botasaurus is a **legitimate improvement** over raw Selenium. The human-like mouse movements via Bézier curves and CDP integration are well-implemented and will help evade basic detection.

However, calling it "undefeatable" is marketing hyperbole. For serious anti-bot systems, you'll still need:
- Residential/mobile proxies
- Real browser profiles with history
- More sophisticated fingerprint management
- Rate limiting and behavioral variation

**Recommendation:** Excellent choice for mid-tier scraping needs where human-like behavior matters. Pair with quality proxies and consider combining with BotBrowser for fingerprint consistency.

**Effectiveness:** Moderate-High | **Complexity:** Low | **Best Use:** Scraper apps with human-like behavior

---

## Resources

- [GitHub Repository](https://github.com/omkarcloud/botasaurus)
- [Documentation](https://github.com/omkarcloud/botasaurus#readme)
- [Bot Detection Tests](https://github.com/omkarcloud/botasaurus/blob/master/bot_detection_tests.py)
