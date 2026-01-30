# Patchright

> A sophisticated binary patching framework for Playwright that bypasses bot detection at the protocol level.

**Repository:** [https://github.com/Kaliiiiiiiiii-Vinyzu/patchright](https://github.com/Kaliiiiiiiiii-Vinyzu/patchright)

**Type:** Browser Automation Framework (Playwright Fork)

**Language Support:** Python, Node.js, .NET

---

## What is Patchright?

Patchright is a patched version of Playwright that modifies the browser driver at compile-time to avoid detection by anti-bot systems. Unlike simple JavaScript injection tools, Patchright fundamentally changes how Playwright communicates with Chrome via the Chrome DevTools Protocol (CDP).

---

## How It Works

### The Core Problem with Playwright

Standard Playwright is easily detected because it uses CDP commands that leave fingerprints:

```javascript
// These CDP calls are detectable via timing attacks and API probing
await client.send('Runtime.enable');
await client.send('Page.addScriptToEvaluateOnNewDocument', {...});
await client.send('Runtime.addBinding', {...});
```

### Patchright's Solution

Patchright applies **22 patches** (~5,856 lines of modifications) to Playwright's source code using AST manipulation. The key innovations:

#### 1. Runtime.enable Bypass

**Traditional Playwright:**
```javascript
await client.send('Runtime.enable');  // Detectable!
```

**Patchright:**
- Never calls `Runtime.enable`
- Uses isolated ExecutionContexts via `Page.createIsolatedWorld`
- Manages execution contexts manually without triggering detection

#### 2. Network-Layer Script Injection

Instead of using detectable CDP commands for script injection, Patchright:

1. Intercepts HTML responses at the network layer
2. Modifies Content-Security-Policy headers
3. Injects `<script>` tags directly into the HTML
4. Scripts self-delete after execution

```html
<!-- Injected as normal HTML - no CDP fingerprint -->
<script nonce="abc123" id="randomId">
  document.getElementById("randomId")?.remove();
  // Your injected code here
</script>
```

#### 3. Command-Line Flag Sanitization

Removes automation-revealing flags:
- `--enable-automation` (removed)
- `--disable-popup-blocking` (removed)
- `--disable-extensions` (removed)
- Adds `--disable-blink-features=AutomationControlled`

---

## Detection Vectors Bypassed

| Detection Method | How Patchright Defeats It |
|---|---|
| `navigator.webdriver` | Disabled via Blink feature flag |
| Runtime.enable timing attacks | Never called - uses isolated contexts |
| CDP binding detection | Bindings added to isolated worlds only |
| Init script fingerprinting | Scripts injected as normal HTML |
| Console API probing | Console.enable disabled entirely |
| CSP violation detection | Headers modified before browser parses |

---

## Services Bypassed

According to the project's claims and community testing:

- Cloudflare
- Akamai
- DataDome
- Kasada
- Shape/F5
- Fingerprint.com
- Bet365
- CreepJS
- Sannysoft
- Incolumitas
- Browserscan
- Pixelscan

---

## Installation

### Python
```bash
pip install patchright
playwright install chromium
```

### Node.js
```bash
npm install patchright
npx playwright install chromium
```

### .NET
```bash
dotnet add package Patchright
```

---

## Usage Example

### Python
```python
from patchright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    page = browser.new_page()
    page.goto("https://bot.sannysoft.com")
    # All tests should pass
    browser.close()
```

### Node.js
```javascript
const { chromium } = require('patchright');

(async () => {
    const browser = await chromium.launch({ headless: false });
    const page = await browser.newPage();
    await page.goto('https://bot.sannysoft.com');
    // All tests should pass
    await browser.close();
})();
```

---

## Pros and Cons

### Advantages

| Advantage | Description |
|---|---|
| Deep Integration | Patches at binary/protocol level, not surface JavaScript |
| Playwright API | Full Playwright API compatibility |
| Active Maintenance | Automatically tracks new Playwright releases |
| Multi-Platform | Windows, Linux, macOS (x64 and ARM64) |
| Multi-Language | Python, Node.js, .NET support |
| Proven Effectiveness | Bypasses major anti-bot services |

### Limitations

| Limitation | Description |
|---|---|
| No Console Output | `console.log()` doesn't work (Console API disabled) |
| Chromium Only | Firefox and WebKit not supported |
| Some Test Failures | Certain Playwright tests fail |
| Extension Issues | Chrome extensions not fully functional |
| Timing Vulnerability | Init scripts have theoretical timing attack vector |

---

## Technical Architecture

### Patch Files Structure

```
driver_patches/
├── chromiumSwitchesPatch.js      # Remove detection flags
├── crPagePatch.js                # Script injection & bindings (~400 lines)
├── crNetworkManagerPatch.js      # Network interception (~370 lines)
├── framesPatch.js                # Avoid Runtime.enable (~250 lines)
├── frameSelectorsPatch.js        # Closed shadow root access
├── javascriptPatch.js            # Execution context switching
├── browserContextPatch.js        # Context initialization
├── pagePatch.js                  # Page-level bindings
└── ... (14 more patches)
```

### Build Process

1. Watches for new Playwright releases
2. Clones Playwright source code
3. Applies all patches via ts-morph (AST manipulation)
4. Builds for all platforms
5. Publishes to GitHub releases and package managers

---

## When to Use Patchright

### Best For:
- Scraping sites protected by Cloudflare, Akamai, DataDome
- Automation requiring full browser capabilities
- Projects already using Playwright
- When you need the full Playwright API

### Not Ideal For:
- Simple scraping (overkill)
- When you need console.log debugging
- Firefox or WebKit automation
- Projects requiring Chrome extensions

---

## Comparison with Alternatives

| Feature | Patchright | Undetected-Chromedriver | Puppeteer-Stealth |
|---|---|---|---|
| Detection Bypass | Excellent | Good | Moderate |
| API | Playwright | Selenium | Puppeteer |
| Approach | Binary patching | Binary patching | JS injection |
| Maintenance | Active | Active | Limited |
| Console.log | No | Yes | Yes |

---

## Related Projects

- [patchright-python](https://github.com/Kaliiiiiiiiii-Vinyzu/patchright-python) - Python bindings
- [patchright-nodejs](https://github.com/AranocuS/patchright-nodejs) - Node.js package
- [patchright-dotnet](https://github.com/DevEnterpriseSoftware/patchright-dotnet/) - .NET package
- [Botright](https://github.com/Kaliiiiiiiiii-Vinyzu/Botright) - Additional fingerprint spoofing layer

---

## Resources

- [GitHub Repository](https://github.com/Kaliiiiiiiiii-Vinyzu/patchright)
- [Playwright Documentation](https://playwright.dev/)
- [Issue Tracker](https://github.com/Kaliiiiiiiiii-Vinyzu/patchright/issues)

---

## Summary

Patchright represents a **protocol-level approach** to browser automation stealth. By modifying how Playwright communicates with Chrome at the CDP layer, it achieves detection bypass that JavaScript injection methods cannot match. The trade-off is losing console output and some edge-case functionality, but for serious anti-bot bypass needs, it's one of the most effective tools available.

**Effectiveness Rating:** High - Bypasses most major anti-bot services

**Complexity:** Low - Drop-in Playwright replacement

**Best Use Case:** Production scraping of heavily protected sites
