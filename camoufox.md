# Camoufox - Deep Technical Analysis

> **Tool Type:** Custom Firefox Build (Anti-Detect Browser)
> **Repository:** [github.com/daijro/camoufox](https://github.com/daijro/camoufox)
> **Approach:** C++ level fingerprint injection + Juggler protocol isolation
> **Effectiveness:** Very High (statistically-accurate fingerprint rotation)
> **Maintenance:** Active development

---

## Table of Contents

- [What is Camoufox?](#what-is-camoufox)
- [How It Works](#how-it-works)
- [Why This Approach is Superior](#why-this-approach-is-superior)
- [Fingerprint Capabilities](#fingerprint-capabilities)
- [Pros and Cons](#pros-and-cons)
- [Installation & Usage](#installation--usage)
- [When to Use](#when-to-use)

---

## What is Camoufox?

Camoufox is a **custom-built Firefox browser** designed specifically for web scraping and automation stealth. Unlike tools that patch or inject into existing browsers, Camoufox compiles Firefox from source with modifications at the C++ implementation level.

**Key differentiator:** All fingerprint spoofing happens in the browser's native C++ code, making it impossible for JavaScript to detect the modifications through property descriptor checks, prototype inspection, or timing analysis.

---

## How It Works

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAMOUFOX ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │  Python Library │───▶│  BrowserForge   │                     │
│  │  (camoufox.py)  │    │  Fingerprints   │                     │
│  └────────┬────────┘    └────────┬────────┘                     │
│           │                      │                               │
│           ▼                      ▼                               │
│  ┌─────────────────────────────────────────┐                    │
│  │         JSON Config Generation          │                    │
│  │   (Statistically accurate profiles)     │                    │
│  └────────────────────┬────────────────────┘                    │
│                       │                                          │
│                       ▼                                          │
│  ┌─────────────────────────────────────────┐                    │
│  │      CUSTOM FIREFOX BUILD               │                    │
│  │  ┌───────────────────────────────────┐  │                    │
│  │  │  C++ Fingerprint Injection        │  │                    │
│  │  │  - Navigator properties           │  │                    │
│  │  │  - Screen/Window dimensions       │  │                    │
│  │  │  - WebGL parameters               │  │                    │
│  │  │  - Audio context                  │  │                    │
│  │  └───────────────────────────────────┘  │                    │
│  │  ┌───────────────────────────────────┐  │                    │
│  │  │  Patched Juggler Protocol         │  │                    │
│  │  │  - Isolated execution scope       │  │                    │
│  │  │  - No page-visible artifacts      │  │                    │
│  │  │  - webdriver = false (C++ level)  │  │                    │
│  │  └───────────────────────────────────┘  │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### C++ Level Fingerprint Injection

The core innovation is modifying Firefox's C++ source code to intercept property getters. From `fingerprint-injection.patch`:

```cpp
// dom/base/nsGlobalWindowInner.cpp
double nsGlobalWindowInner::GetInnerWidth(ErrorResult& aError) {
  // Camoufox intercepts BEFORE the original implementation
  if (auto value = MaskConfig::GetDouble("window.innerWidth"))
    return value.value();  // Return spoofed value
  // Fall through to original only if no spoof configured
  FORWARD_TO_OUTER_OR_THROW(GetInnerWidthOuter, (aError), aError, 0);
}

int32_t nsGlobalWindowInner::GetScreenX(CallerType aCallerType,
                                        ErrorResult& aError) {
  if (auto value = MaskConfig::GetInt32("window.screenX"))
    return value.value();  // Spoofed screen position
  FORWARD_TO_OUTER_OR_THROW(GetScreenXOuter, (aCallerType, aError), aError, 0);
}
```

**Why this is undetectable:**
- The getter returns the spoofed value directly from C++
- There's no JavaScript wrapper or proxy
- `Object.getOwnPropertyDescriptor()` shows a native function
- `toString()` returns `[native code]`
- No timing difference between real and spoofed values

### Juggler Protocol Isolation

Camoufox uses **Juggler** (Firefox's automation protocol) instead of CDP. It patches Juggler to run in an isolated scope:

```
┌─────────────────────────────────────────────────────────────┐
│                    PAGE EXECUTION                            │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │   PAGE SCOPE         │  │   JUGGLER SCOPE      │         │
│  │   (Visible to JS)    │  │   (Isolated)         │         │
│  │                      │  │                      │         │
│  │  - User's website    │  │  - Playwright code   │         │
│  │  - Detection scripts │  │  - Element queries   │         │
│  │  - Anti-bot checks   │  │  - Script injection  │         │
│  │                      │  │  - Event listeners   │         │
│  │  ❌ Cannot see       │  │                      │         │
│  │     Juggler scope    │  │  ✅ Can read/modify  │         │
│  │                      │  │     page scope       │         │
│  └──────────────────────┘  └──────────────────────┘         │
│                                                              │
│  🔒 Complete isolation - no __playwright__ variables leak   │
└─────────────────────────────────────────────────────────────┘
```

From `1-leak-fixes.patch` - the webdriver property fix at C++ level:

```cpp
// dom/base/Navigator.cpp
bool Navigator::Webdriver() {
  // Camoufox: simply return false at the C++ level
  // No JavaScript patching needed
  return false;
}
```

### BrowserForge Integration

Camoufox uses **BrowserForge** to generate statistically accurate fingerprints:

```python
# pythonlib/camoufox/fingerprints.py
from browserforge.fingerprints import FingerprintGenerator

FP_GENERATOR = FingerprintGenerator(
    browser='firefox',
    os=('linux', 'macos', 'windows')
)
```

This means:
- ~5% of fingerprints will be Linux (matching real market share)
- Screen resolutions follow actual usage distribution
- Hardware combinations are realistic (no Windows + M1 GPU)

---

## Why This Approach is Superior

### vs. JavaScript Injection (puppeteer-extra-stealth, etc.)

| Detection Method | JS Injection | Camoufox |
|-----------------|--------------|----------|
| `Object.getOwnPropertyDescriptor` check | ❌ Detectable | ✅ Native |
| `Function.toString()` returns `[native code]` | ❌ Often fails | ✅ Always |
| Worker vs main context mismatch | ❌ Detectable | ✅ Consistent |
| Property access timing analysis | ❌ Delay | ✅ Native speed |

### vs. CDP-based Solutions (Playwright/Chrome, Puppeteer)

| Detection Method | CDP-based | Camoufox |
|-----------------|-----------|----------|
| `navigator.webdriver` | Patchable but leaky | ✅ C++ level false |
| `window.__playwright__` | ❌ Present | ✅ Isolated scope |
| Runtime.enable detection | ❌ Detectable | ✅ Uses Juggler |

---

## Fingerprint Capabilities

### Full Coverage

| Category | Properties Spoofed |
|----------|-------------------|
| **Navigator** | userAgent, platform, vendor, hardwareConcurrency, deviceMemory, languages |
| **Screen** | width, height, availWidth, availHeight, colorDepth, pixelDepth |
| **Window** | innerWidth/Height, outerWidth/Height, screenX/Y, devicePixelRatio |
| **WebGL** | Vendor, renderer, extensions, shader formats, max textures |
| **Audio** | sampleRate, outputLatency, maxChannelCount, voices |
| **Network** | WebRTC IP (protocol level), User-Agent header, Accept-Language |
| **Other** | Geolocation, timezone, locale, battery, media devices, fonts |

---

## Pros and Cons

### Advantages

| Pro | Details |
|-----|---------|
| **C++ level spoofing** | Completely native, undetectable via JS |
| **Juggler isolation** | No page-visible automation artifacts |
| **Statistical accuracy** | BrowserForge mimics real traffic patterns |
| **Comprehensive** | Navigator, Screen, WebGL, Audio, WebRTC, etc. |
| **Active development** | Regular updates |
| **Human-like mouse** | Built-in natural movement algorithm |
| **Debloated** | ~200MB less memory than standard Firefox |

### Disadvantages

| Con | Details |
|-----|---------|
| **Firefox only** | Can't impersonate Chrome (~65% market share) |
| **SpiderMonkey detection** | Some WAFs detect Firefox engine specifically |
| **Build complexity** | Requires Linux to build from source |
| **Market share** | Firefox ~3% may flag on some systems |

---

## Installation & Usage

### Quick Start

```bash
pip install camoufox
playwright install firefox
```

```python
# Sync API
from camoufox.sync_api import Camoufox

with Camoufox() as browser:
    page = browser.new_page()
    page.goto("https://example.com")
```

```python
# Async API
from camoufox.async_api import AsyncCamoufox

async with AsyncCamoufox() as browser:
    page = await browser.new_page()
    await page.goto("https://example.com")
```

### Custom Fingerprint

```python
config = {
    "window.innerWidth": 1920,
    "window.innerHeight": 1080,
    "navigator.platform": "Win32",
}

with Camoufox(config=config) as browser:
    page = browser.new_page()
```

### With Proxy + Auto Geolocation

```python
with Camoufox(
    proxy={"server": "http://proxy:8080"},
    geoip=True,  # Auto-detect geo from proxy IP
) as browser:
    page = browser.new_page()
```

---

## When to Use

### Recommended For

- Firefox-tolerant targets
- High stealth requirements (JS injection fails)
- Fingerprint rotation needs
- Python projects
- Long-running sessions

### Not Recommended For

- Chrome-only sites
- SpiderMonkey-detecting WAFs
- Quick prototyping (heavier setup)
- Non-Python projects (server mode needed)

---

## Key Files

| File | Purpose |
|------|---------|
| `patches/fingerprint-injection.patch` | C++ hooks for all values |
| `patches/playwright/1-leak-fixes.patch` | webdriver=false, isolation |
| `patches/webgl-spoofing.patch` | WebGL parameter interception |
| `patches/webrtc-ip-spoofing.patch` | Protocol-level IP spoofing |
| `pythonlib/camoufox/fingerprints.py` | BrowserForge integration |

---

## Comparison

| Feature | Camoufox | XDriver | Patchright | puppeteer-stealth |
|---------|:--------:|:-------:|:----------:|:-----------------:|
| Spoofing Level | C++ | CDP | Binary | JavaScript |
| Browser | Firefox | Chromium | Chromium | Chromium |
| Fingerprint Rotation | ✅ Full | ❌ None | Partial | Partial |
| Automation Isolation | ✅ Complete | ✅ Good | ✅ Good | ❌ Partial |
| Detection Difficulty | Very Hard | Hard | Hard | Medium |

---

## Conclusion

Camoufox represents the **gold standard for Firefox-based anti-detection**. By modifying Firefox at the C++ level, it achieves truly native-appearing fingerprint spoofing that JavaScript inspection cannot detect.

**Best for:** Maximum stealth when target doesn't discriminate against Firefox users.

**Limitation:** Firefox's ~3% market share inherently makes you a minority browser user.

---

*Analysis conducted for educational purposes. Use responsibly.*
