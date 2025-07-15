---
title: "One by One – LA CTF 2024"  
date: 2024-02-18  
category: Misc (Google Forms brute-forcing)  
platform: LA CTF 2024  
author: Vaishnav Baraskar
tags: ["CTF", "LA CTF", "Misc", "Google Forms", "Brute Force", "Automation", "Web Abuse", "CTF 2024", "Challenge Writeup", "Vaishnav Baraskar"]
---


## 0x00 – Prologue

Brute-forcing a Google Form? Yeah, it sounds dumb until you realize the form is leaking state via some sneaky HTML fields. That's when it turns into an actual side-channel attack and not just clicking buttons like a bot. This was one of those problems where you stare at Chrome DevTools long enough, and suddenly you're deep in Puppeteer automations and page parity logic.

The challenge was called "One by One" — subtle hint at what was to come. Letter-by-letter flag extraction.

No crypto. No binary. Just me, a browser, and a form that thought it could hide the truth behind a few JavaScript layers.

---

## 0x01 – The Form

The interface looked like any generic Google Form. Typical UI. One input box.

Submit the flag? Sure.

There was no feedback on screen, no error messages. But when I popped open DevTools, I saw this little input field embedded in the form submission:

```html
<input name="pageHistory" type="hidden" value="0,1,3">
```

Hmm. That `pageHistory` array changed every time I submitted something.

When I tried a random string:

```txt
pageHistory: 0,1,3,5,7
```

When I guessed the flag correctly (up to a certain prefix), it looked like this:

```txt
pageHistory: 0,2,4,6
```

See it?

- Correct guesses increment by **even** numbers.
- Wrong guesses lead to **odd** page transitions.

Looks like the form was branching on correctness internally and recording progress by modifying the page navigation flow.

This wasn’t just a form. It was a finite state machine. And I was about to brute-force my way through its every node.

---

## 0x02 – The Attack Plan

So here was the plan:

1. Start with an empty flag.
2. For each position, guess a character.
3. Submit it.
4. Check `pageHistory` length or last number.
5. If it ended on an even page, the character was right.
6. Move to next character.

The key was using a headless browser, because Google Forms isn't friendly with raw curl or requests. I fired up Puppeteer.

---

## 0x03 – The Puppeteer Script

Here’s the stripped-down version of my script:

```js
const puppeteer = require('puppeteer');

(async () => {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789{}_!"\',.*';
  let known = 'lactf{';

  const browser = await puppeteer.launch({ headless: true });
  const page = await browser.newPage();

  while (!known.endsWith('}')) {
    for (const c of chars) {
      const guess = known + c;

      await page.goto('https://docs.google.com/forms/d/e/1FAIpQLS.../viewform');

      // Fill input
      await page.type('input[type="text"]', guess);
      await page.click('div[role="button"]');
      await page.waitForTimeout(1500);  // wait for page to process

      const pageHistory = await page.$eval('input[name="pageHistory"]', el => el.value);

      const historyArray = pageHistory.split(',').map(Number);
      const last = historyArray[historyArray.length - 1];

      if (last % 2 === 0) {
        known += c;
        console.log('Correct char:', c);
        break;
      }
    }
  }

  console.log('Final flag:', known);
  await browser.close();
})();
```

This ran slower than I'd like, thanks to Google Forms rate-limits and page transitions. But hey, it got the job done.

---

## 0x04 – What Worked (and What Didn't)

At first, I tried guessing multiple characters at once. Big mistake. If you submit too long of a wrong prefix, the form gives up and kicks you to a dead end.

Then I thought: maybe it's detecting automation? So I added delays, randomized typing, and even used a non-headless browser to verify parity detection wasn’t blocked.

Eventually I found that Google doesn’t validate aggressively if you:

- use proper user-agent
- wait between interactions
- and don’t flood too many requests in parallel

---

## 0x05 – Flag Reconstructed

After many iterations and lots of noisy page transitions, the full flag came out like this:

```
lactf{1_by_0n3_by3_un0_*,"g1'}
```

Strange format, but valid. The challenge name now made total sense.

---

## 0x06 – Takeaways

- Hidden fields leak more than you'd think.
- Side-channels can live inside page navigations.
- Puppeteer is a beast for automating weird stuff.
- Think like a parser: if the app can branch, you can trace it.

One character at a time might be slow, but it’s precise. And in CTFs, precision > speed.

---

## 0x07 – Tools Used

- Puppeteer (Node.js)
- Chrome DevTools
- Regex, console.log, coffee

---

## 0x08 – Flag

```
lactf{1_by_0n3_by3_un0_*,"g1'}
```

---

