# XDriver - Technical Analysis

> **Repository:** [nicebots-xyz/x_driver](https://github.com/nicebots-xyz/x_driver)
> **Category:** Playwright Stealth Patcher
> **Language:** Python
> **Type:** Runtime patching of Playwright driver files

---

## Overview

XDriver is a stealth patching tool that modifies Playwright's driver at the binary/JavaScript level to evade bot detection. Unlike script-injection approaches, it directly replaces Playwright's core files with hardened versions.

## Verdict

| Aspect | Rating | Notes |
|--------|--------|-------|
| **Overall Quality** | ⭐⭐⭐⭐ | Well-engineered patching approach |
| **Anti-Detection** | ⭐⭐⭐⭐ | Comprehensive CDP leak prevention |
| **Ease of Use** | ⭐⭐⭐⭐⭐ | One-command activation |
| **Maintenance Risk** | ⭐⭐ | Version-locked to Playwright 1.52.0 |
| **Documentation** | ⭐⭐⭐ | Basic but functional |

**TL;DR:** Convenient one-command stealth for Playwright. Great for quick testing, but version lock is a significant limitation for production use.

---

## How It Works - Technical Deep Dive

### Two-Stage Patching Mechanism

XDriver uses a fundamentally different approach than other tools - it **replaces Playwright's driver files entirely** rather than injecting scripts or wrapping APIs.

### Stage 1: File Replacement

```bash
x_driver activate
```

This command performs:

```
Original Playwright:
playwright/
├── driver/
│   └── package/           ← Original CDP implementation
│       ├── crConnection.js
│       ├── crPage.js
│       ├── browserContext.js
│       └── frames.js
└── __init__.py

After Activation:
playwright/
├── driver/
│   ├── package/           ← Patched (from bundles/)
│   │   ├── crConnection.js    [PATCHED]
│   │   ├── crPage.js          [PATCHED]
│   │   ├── browserContext.js  [PATCHED]
│   │   └── frames.js          [PATCHED]
│   └── package_1/         ← Backup of original
└── __init__.py            [MODIFIED]
```

**Why this approach:**
- Modifies at the source level - no runtime overhead
- Changes persist across sessions
- No code changes required in user scripts
- Completely transparent to existing Playwright code

### Stage 2: CDP Leak Prevention

The patched files implement multiple anti-detection techniques:

#### A. CDP Connection Cloaking (`crConnection.js`)

```javascript
// Original Playwright exposes CDP connection artifacts
// XDriver patches to hide these

class CRConnection {
    constructor(transport, protocolLogger, browserLogsCollector) {
        // PATCHED: Remove identifiable session IDs
        this._sessions = new Map();

        // PATCHED: Intercept and sanitize CDP messages
        this._transport = new PatchedTransport(transport);
    }

    // PATCHED: Hide connection from page scripts
    _onMessage(message) {
        // Filter out detectable CDP artifacts
        if (this._shouldHideMessage(message)) {
            return;
        }
        // ... handle message
    }
}
```

#### B. Script Injection Concealment (`crPage.js`)

```javascript
// Original Playwright leaks init script artifacts:
// - window.__pwInitScripts
// - __playwright__binding__
// - Page.addScriptToEvaluateOnNewDocument traces

// XDriver patches:
async _addInitScript(source) {
    // PATCHED: Remove script identification markers
    const sanitizedSource = this._removePlaywrightMarkers(source);

    // PATCHED: Use isolated execution context
    await this._client.send('Page.addScriptToEvaluateOnNewDocument', {
        source: sanitizedSource,
        worldName: '__playwright_utility_world__'  // Isolated from page
    });
}

_removePlaywrightMarkers(source) {
    // Remove identifiable patterns
    return source
        .replace(/__playwright/g, '_' + randomId())
        .replace(/__pwInitScripts/g, '_' + randomId());
}
```

#### C. Developer Tools Evasion (`browserContext.js`)

```javascript
// Detection scripts check for DevTools/Inspector access
// XDriver patches to hide these signals

class BrowserContext {
    async _initialize() {
        // PATCHED: Don't expose Runtime.enable
        // await this._client.send('Runtime.enable');  // REMOVED

        // PATCHED: Use isolated contexts instead
        await this._setupIsolatedContexts();
    }

    async _setupIsolatedContexts() {
        // Create utility world without Runtime.enable
        await this._client.send('Page.createIsolatedWorld', {
            frameId: this._frameId,
            worldName: '__utility__',
            grantUniveralAccess: true
        });
    }
}
```

#### D. Binding Exposure Prevention

```javascript
// Original Playwright exposes:
// - window.__pwBinding
// - exposeFunctionLeak vulnerabilities

// XDriver patches PageBinding class:
class PageBinding {
    async add(name, callback) {
        // PATCHED: Add to isolated world only
        await this._context._client.send('Runtime.addBinding', {
            name: this._obfuscateName(name),
            executionContextName: '__playwright_utility_world__'
        });
    }

    _obfuscateName(name) {
        // Randomize binding names
        return '_' + crypto.randomBytes(8).toString('hex');
    }
}
```

#### E. WebRTC Leak Protection

```javascript
// Prevents WebRTC from leaking real IP addresses
async _patchWebRTC(page) {
    await page.addInitScript(() => {
        // Override RTCPeerConnection
        const originalRTCPeerConnection = window.RTCPeerConnection;
        window.RTCPeerConnection = function(...args) {
            const pc = new originalRTCPeerConnection(...args);
            // Filter ICE candidates to prevent IP leakage
            const originalAddIceCandidate = pc.addIceCandidate.bind(pc);
            pc.addIceCandidate = function(candidate) {
                if (candidate && candidate.candidate) {
                    // Filter local IP addresses
                    if (candidate.candidate.includes('.local')) {
                        return Promise.resolve();
                    }
                }
                return originalAddIceCandidate(candidate);
            };
            return pc;
        };
    });
}
```

#### F. Service Worker Interception

```javascript
// Block service worker registration that could detect automation
async _blockServiceWorkers(page) {
    await page.addInitScript(() => {
        // Override navigator.serviceWorker.register
        if (navigator.serviceWorker) {
            navigator.serviceWorker.register = () => {
                return Promise.reject(new Error('Service workers disabled'));
            };
        }
    });
}
```

---

## Architecture

```
XDriver Patching Flow:
┌────────────────────────────────────────────────────────┐
│  pip install x_driver                                  │
└────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│  x_driver activate                                     │
│  ┌──────────────────────────────────────────────────┐ │
│  │ 1. Backup: /driver/package → /driver/package_1   │ │
│  │ 2. Replace: /bundles/package → /driver/package   │ │
│  │ 3. Modify: playwright/__init__.py                │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│  from playwright.async_api import async_playwright     │
│  # Uses patched driver automatically                   │
└────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│  x_driver deactivate                                   │
│  # Restores original files from backup                 │
└────────────────────────────────────────────────────────┘
```

---

## Services Bypassed

According to the project's performance claims:

### Enterprise Anti-Bot
| Service | Status |
|---------|--------|
| Cloudflare WAF | ✅ |
| Cloudflare Turnstile | ✅ |
| Kasada | ✅ |
| DataDome | ✅ |
| PerimeterX | ✅ |
| Imperva | ✅ |
| Fingerprint.com | ✅ |

### Fingerprinting Tests
| Test | Result |
|------|--------|
| CreepJS | 100% Anonymous |
| BrowserScan | 87% |
| Rebrowser Bot Detector | All tests pass |
| IP-API Bot Detection | Pass |
| IPQualityScore | Pass |
| Whoer.net | High Anonymity |
| AmIUnique | No unique fingerprint |
| Cover Your Tracks (EFF) | Strong protection |

---

## Usage

### Installation & Activation
```bash
pip install x_driver
pip install playwright==1.52.0  # Version-locked!
playwright install chromium

# Activate patching
x_driver activate
```

### Basic Usage (No Code Changes)
```python
from playwright.async_api import async_playwright

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)
        page = await browser.new_page()

        # All requests now use patched driver
        await page.goto("https://bot.sannysoft.com")
        # Tests should pass

        await browser.close()

import asyncio
asyncio.run(main())
```

### Deactivation
```bash
x_driver deactivate  # Restores original Playwright
```

---

## Pros and Cons

### Advantages

| Advantage | Details |
|-----------|---------|
| **One-Command Activation** | `x_driver activate` - done |
| **No Code Changes** | Existing Playwright scripts work as-is |
| **Comprehensive Patching** | CDP, bindings, WebRTC, service workers |
| **Reversible** | Clean deactivation restores original |
| **Wide Detection Coverage** | Passes major anti-bot tests |
| **Open Source** | Apache 2.0 license |
| **Transparent** | User doesn't need to understand internals |

### Limitations

| Limitation | Details |
|------------|---------|
| **Version Lock** | Only works with Playwright 1.52.0 |
| **Invasive Approach** | Directly modifies library files |
| **Single Author** | Limited community support |
| **Beta Status** | v1.0.1 with limited production track record |
| **Chromium Only** | No Firefox/WebKit support |
| **Backup Dependency** | Corrupted backup = manual recovery |
| **Empty Examples** | Example files in repo are empty |

---

## Comparison with Patchright

| Feature | XDriver | Patchright |
|---------|:-------:|:----------:|
| **Patching Approach** | Runtime file replacement | Compile-time source patching |
| **Installation** | `pip install` + activate | Separate package (`pip install patchright`) |
| **Version Flexibility** | Locked to 1.52.0 | Tracks Playwright releases |
| **Reversibility** | Easy `deactivate` | N/A (separate package) |
| **Code Changes** | None required | Import from `patchright` instead |
| **Maintenance** | Single author | Active community |
| **Maturity** | Beta (v1.0.1) | Established |
| **Multi-Language** | Python only | Python, Node.js, .NET |

### When to Choose XDriver over Patchright

**Choose XDriver if:**
- You have existing Playwright code you can't modify
- You need quick testing without code changes
- You're okay with version lock
- You want easy on/off switching

**Choose Patchright if:**
- You need version flexibility
- You're starting a new project
- You need Node.js or .NET support
- You want active community support

---

## When to Use XDriver

### Best For:
- Quick Playwright stealth **without code changes**
- Testing against multiple anti-bot services
- Existing Playwright codebases
- Rapid prototyping

### Not Ideal For:
- Production systems needing **version flexibility**
- Projects requiring **Firefox/WebKit**
- Long-term maintenance concerns
- Teams needing community support

---

## Bottom Line

XDriver offers **convenient one-command stealth** for Playwright with comprehensive CDP leak prevention. The trade-off is being locked to a specific Playwright version and relying on invasive file modifications.

**Good for:** Quick testing and existing codebases where you can't modify imports.

**Consider Patchright instead:** For production use, version flexibility, or new projects.

**Effectiveness:** High | **Complexity:** Very Low | **Best Use:** Quick Playwright stealth testing

---

## Resources

- [GitHub Repository](https://github.com/nicebots-xyz/x_driver)
- [PyPI Package](https://pypi.org/project/x-driver/)
- [Playwright Documentation](https://playwright.dev/python/)
