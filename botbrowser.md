# BotBrowser - Technical Analysis

> **Repository:** [botswin/BotBrowser](https://github.com/botswin/BotBrowser)
> **Category:** Patched Chromium Browser / Anti-Fingerprinting Engine
> **Type:** Modified Chromium binary (not a wrapper/library)
> **Last Analyzed:** January 2026

## Overview

BotBrowser is a **patched Chromium browser** designed to prevent browser fingerprinting. Unlike wrapper libraries (Botasaurus) or driver patches (Patchright, XDriver), BotBrowser is a fully modified Chromium binary with proprietary anti-fingerprinting technology baked directly into the browser engine.

## Verdict

| Aspect | Rating | Notes |
|--------|--------|-------|
| **Overall Quality** | ⭐⭐⭐⭐⭐ | Enterprise-grade engineering |
| **Anti-Detection** | ⭐⭐⭐⭐⭐ | Most comprehensive solution analyzed |
| **Ease of Use** | ⭐⭐⭐ | Requires understanding of profiles/flags |
| **Documentation** | ⭐⭐⭐⭐ | Extensive but technical |
| **Cost** | ⭐⭐ | Free tier limited, enterprise features require subscription |

**TL;DR:** The most sophisticated anti-detection browser available. Overkill for simple scraping, essential for enterprise-level evasion needs.

---

## What Makes It Different

### It's NOT a wrapper or patch - it's a custom browser

| Tool | Architecture |
|------|--------------|
| Botasaurus | Python wrapper around Selenium |
| Patchright | Patched Playwright driver |
| XDriver | Patched Playwright CDP layer |
| **BotBrowser** | **Fully modified Chromium binary** |

BotBrowser modifies Chromium at the source level across multiple subsystems:
- V8 JavaScript engine
- Blink rendering engine
- Skia graphics library
- HarfBuzz text shaping
- Network stack
- GPU process

---

## How It Works

### Core Philosophy: Fingerprint Standardization

BotBrowser doesn't try to "hide" that it's automated. Instead, it makes the automation signal **identical across all platforms and sessions**:

```
Windows + Profile A = Fingerprint X
macOS   + Profile A = Fingerprint X
Linux   + Profile A = Fingerprint X
```

This prevents tracking systems from correlating users across sessions.

---

### 1. Encrypted Profile System

Fingerprints are stored in encrypted `.enc` profile files containing:

```
Profile Contents:
├── Browser identification (brand, version, fullVersionList)
├── Device properties (screen resolution, DPI, color depth)
├── Platform info (Windows/macOS/Linux/Android)
├── Font availability lists
├── Hardware specs (CPU cores, RAM, GPU)
├── WebGL/Canvas parameters
├── WebRTC ICE server presets
├── Media codec lists
└── Plugin information
```

**Usage:**
```bash
chrome.exe --bot-profile="path/to/profile.enc" --user-data-dir="$(mktemp -d)"
```

---

### 2. Cross-Platform Font Parity

One of the biggest fingerprinting vectors is font enumeration. BotBrowser solves this by:

- **Embedding font bundles** for Windows, macOS, Linux, and Android
- DOM text rendering uses **only bundled fonts** (never exposes host OS fonts)
- **HarfBuzz text shaping** tuned for identical metrics across platforms
- **Skia anti-aliasing** standardized for consistent Canvas rendering

**Result:** Text measurement APIs return identical values regardless of host OS.

---

### 3. Deterministic Canvas/WebGL Noise

Instead of random noise (easily detected), BotBrowser uses **deterministic noise**:

```javascript
// Same session = same noise (repeatable)
// Different seed = different noise (profiles vary)
```

Applied to:
- Canvas 2D pixel data
- WebGL texture hashes
- WebGPU buffer output
- AudioContext fingerprint

**Key property:** Noise is reproducible per-session but varies across profiles, preventing hash-based correlation.

---

### 4. CDP Leak Prevention

Standard automation frameworks leak artifacts:

```javascript
// Playwright leaks these:
window.__playwright__binding__
window.__pwInitScripts

// Puppeteer leaks:
window.__puppeteer_evaluation_script__
```

BotBrowser runs automation in an **isolated execution context** that's invisible to page JavaScript. The `--bot-script` flag enables framework-less automation with privileged `chrome.debugger` access.

---

### 5. Network-Level Protection

| Feature | Free | PRO | ENT |
|---------|:----:|:---:|:---:|
| Proxy support | ✅ | ✅ | ✅ |
| SOCKS5H (DNS in tunnel) | ✅ | ✅ | ✅ |
| Per-context proxy | ❌ | ❌ | ✅ |
| UDP-over-SOCKS5 | ❌ | ❌ | ✅ |
| Local DNS solver | ❌ | ❌ | ✅ |

**UDP-over-SOCKS5** is unique - tunnels QUIC and WebRTC STUN traffic through the proxy, preventing IP leakage that other tools miss.

---

### 6. Headless/GUI Parity

Most tools have detectable differences between headless and GUI modes. BotBrowser ensures:

- Identical GPU signals in both modes
- Same WebGPU behavior
- Matching media capabilities
- Consistent timing behavior

**Result:** Headless automation is indistinguishable from GUI usage.

---

## Chromium Patches (Examples)

### removeHeadless.diff
Removes "Headless" from User-Agent string in headless mode:
```diff
- user_agent = "HeadlessChrome/" + version;
+ user_agent = "Chrome/" + version;
```

### timezone.diff
Intercepts ICU timezone initialization:
```cpp
// Instead of system timezone:
icu::TimeZone* tz = BotProfile::Get("variables.timezone");
```

### webglAttrs.diff
Overrides WebGL context attributes:
```cpp
WebGLContextAttributes GetContextAttributes() {
  return BotProfile::GetWebGLAttributes();
}
```

---

## Per-Context Fingerprint (ENT Tier3)

The most advanced feature - run 50 different fingerprints in a **single browser process**:

```javascript
// Each context gets independent fingerprint
await cdp.send('BotBrowser.setBrowserContextFlags', {
  contextId: '<context-id>',
  botbrowserFlags: {
    'bot-profile': '/path/to/profile-A.enc',
    'bot-config-timezone': 'America/New_York',
  }
});

// Context B has completely different identity
await cdp.send('BotBrowser.setBrowserContextFlags', {
  contextId: '<context-id-B>',
  botbrowserFlags: {
    'bot-profile': '/path/to/profile-B.enc',
    'bot-config-timezone': 'Europe/London',
  }
});
```

**Benefit:** 50x less memory than launching 50 separate browser instances.

---

## Detection Systems Bypassed

BotBrowser claims validation against **31+ anti-bot systems**:

### CAPTCHA Systems
- Cloudflare (Turnstile, Challenge, WAF)
- Google reCAPTCHA v2/v3
- hCaptcha
- FunCaptcha
- GeeTest
- Kasada

### Bot Detection Platforms
- Akamai/Kona
- DataDome
- PerimeterX
- Imperva (Incapsula)
- F5 Shape
- ThreatMetrix

### Fingerprinting Services
- FingerprintJS Pro
- CreepJS
- BrowserScan
- Pixelscan

### E-Commerce Platforms
- Temu, Shopee, Naver
- Walmart, Nike, Ticketmaster
- Instagram, TikTok

**Note:** Claims are backed by 50,000+ test sessions, but anti-bot is an arms race - results may vary.

---

## Integration Examples

### Playwright
```javascript
const { chromium } = require('playwright');

const browser = await chromium.launch({
  executablePath: '/path/to/BotBrowser/chrome',
  args: [
    '--bot-profile=/path/to/profile.enc',
    '--bot-config-timezone=auto'
  ]
});

const page = await browser.newPage();
// Remove framework artifacts
await page.addInitScript(() => {
  delete window.__playwright__binding__;
  delete window.__pwInitScripts;
});

await page.goto('https://example.com');
```

### Puppeteer
```javascript
const puppeteer = require('puppeteer');

const browser = await puppeteer.launch({
  executablePath: '/path/to/BotBrowser/chrome',
  args: [
    '--bot-profile=/path/to/profile.enc',
    '--proxy-server=socks5h://user:pass@proxy:1080'
  ]
});
```

### Bot-Script (Framework-less)
```javascript
// script.js - runs with privileged chrome.debugger access
chrome.debugger.attach({targetId}, "1.3", () => {
  chrome.debugger.sendCommand(targetId, "Page.navigate", {
    url: "https://example.com"
  });
});
```

```bash
chrome.exe --bot-profile=profile.enc --bot-script=script.js
```

---

## Key CLI Flags

### Fingerprint Control
```bash
--bot-profile="/path/to/profile.enc"      # Required: encrypted profile
--bot-config-timezone="auto"              # Auto-detect from proxy IP
--bot-config-languages="en-US,en"         # Browser languages
--bot-config-noise-canvas=true            # Enable canvas noise
--bot-config-noise-webgl-image=true       # Enable WebGL noise
--bot-noise-seed=1.05                     # Deterministic noise (ENT)
```

### Network
```bash
--proxy-server=socks5h://user:pass@proxy:1080  # DNS in tunnel
--bot-webrtc-ice="google"                      # ICE server preset
--bot-local-dns                                # Local DNS (ENT)
```

### Automation
```bash
--bot-script="script.js"                  # Framework-less automation
--bot-always-active=true                  # Fake active window (PRO)
--bot-inject-random-history               # Fake browsing history (PRO)
```

---

## Pricing Tiers

| Feature | Free | PRO | ENT T1 | ENT T2 | ENT T3 |
|---------|:----:|:---:|:------:|:------:|:------:|
| Demo profiles | ✅ | ✅ | ✅ | ✅ | ✅ |
| Android emulation | ❌ | ✅ | ✅ | ✅ | ✅ |
| Per-context proxy | ❌ | ❌ | ✅ | ✅ | ✅ |
| Noise seed control | ❌ | ❌ | ❌ | ✅ | ✅ |
| Per-context fingerprint | ❌ | ❌ | ❌ | ❌ | ✅ |
| UDP-over-SOCKS5 | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## Limitations

### What It Does NOT Handle

1. **Server-side behavioral analysis**
   - Bot detection based on mouse patterns, click sequences
   - Requires additional behavioral simulation (like Botasaurus)

2. **Account-level detection**
   - New account patterns, unusual activity
   - No amount of fingerprint spoofing helps here

3. **Network-level detection beyond TLS**
   - Deep packet inspection by ISPs
   - Traffic pattern analysis

4. **Future detection updates**
   - Anti-bot systems constantly evolve
   - Bypasses may break with updates

5. **Legal/ToS compliance**
   - Tool doesn't make scraping legal
   - User responsibility for compliance

---

## Comparison to Alternatives

| Feature | BotBrowser | Patchright | XDriver | Botasaurus |
|---------|:----------:|:----------:|:-------:|:----------:|
| **Cross-platform parity** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐ |
| **Fingerprint control** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **CDP leak prevention** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **Network privacy** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| **Human simulation** | ⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Ease of use** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Cost** | $$ | Free | Free | Free |
| **Open source** | Partial | Partial | No | Yes |

---

## When to Use BotBrowser

### Good For
- Enterprise-level anti-bot bypass
- Multi-account management at scale
- Cross-platform fingerprint consistency
- Long-running automation requiring stability
- Scenarios where memory efficiency matters (per-context fingerprint)

### Overkill For
- Simple scraping without bot protection
- One-off data collection
- Projects with limited budget
- Scenarios where Patchright/XDriver suffice

### Not Suitable For
- Behavioral detection bypass alone (combine with Botasaurus)
- Legal compliance automation (your responsibility)
- Real-time high-frequency scraping (still detectable)

---

## Bottom Line

BotBrowser is the **most comprehensive anti-fingerprinting solution** in this analysis. It operates at a fundamentally different level than wrapper libraries - modifying Chromium's core to standardize fingerprints across platforms.

**Key innovations:**
1. Same profile = identical fingerprint on Windows/macOS/Linux
2. Embedded cross-platform fonts (no host OS leakage)
3. Deterministic noise (reproducible but varied)
4. Per-context fingerprinting (50 identities, 1 process)
5. UDP-over-SOCKS5 (WebRTC/QUIC tunneling)

**The catch:** Subscription pricing and learning curve. For serious enterprise use, it's worth it. For casual scraping, free alternatives may suffice.

**Recommendation:** Combine with Botasaurus for human-like behavior + BotBrowser for fingerprint consistency = maximum evasion.

---

## Links

- [GitHub Repository](https://github.com/botswin/BotBrowser)
- [Documentation](https://github.com/botswin/BotBrowser#readme)
- [Pricing/Tiers](https://github.com/botswin/BotBrowser#licensing)
