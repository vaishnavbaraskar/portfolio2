---
title: "Reverse Prophecy: Unraveling the Magic 8 Ball – Flare-On 2022"
description: "Dissecting a deceptive 32-bit Magic 8 Ball binary from Flare-On. Static analysis, strcmp chains, x32dbg, and the creative chaos of reverse engineering."
date: '2022-11-11'
categories: ["Reverse Engineering", "CTF Writeups"]
tags: ["Flare-On", "x86", "Static Analysis", "Dynamic Debugging", "Ghidra", "Binary Patching"]
header:
  image: "/images/flareon-magic8ball.jpg"
---

# **Prologue — Binary Fortune Telling**

It was deep into the evening when I first ran the Magic 8 Ball binary. The kind of binary that greets you with a friendly UI — asking questions like it’s all innocent. But in the back of my head, I knew this wasn’t just a game of chance.

It was Flare-On 2022. And this wasn’t a toy. It was a puzzle waiting to be dissected.

---

# **Stage 1 — Getting the Lay of the Binary**

The file: `magic8ball.exe`  
Type: 32-bit PE executable

```bash
file magic8ball.exe
```

Output confirmed what I expected:

```
PE32 executable (GUI) Intel 80386, for MS Windows
```

I loaded it into a sandboxed Windows VM and ran it. It asked for input. I typed something. It returned:

```
"Outlook not so good."
```

Cheeky.

But I knew the real answer — the **flag** — was hidden behind layers of conditions.

---

# **Stage 2 — Diving Into Static Analysis (Ghidra)**

I fired up Ghidra, loaded the binary, and let it churn through the functions. There was a point early in the main function where several `strcmp` calls were chained together — suspiciously validating user input.

Here’s a pseudo-representation of what I was looking at:

```c
if (strcmp(user_input, "quantum") == 0) {
    if (strcmp(user_input + 3, "ntum") == 0) {
        // more nested checks
        ...
    }
}
```

It was a mess of nested `strcmp` calls. Definitely not meant to be brute-forced.

---

# **Stage 3 — Dynamic Debugging (x32dbg)**

Next stop: x32dbg.

I set breakpoints on `strcmp` and ran the binary. Every time it hit a `strcmp`, I observed the input and the comparison string.

Here’s what I used:

```assembly
bp kernel32!strcmp
```

With each breakpoint hit, I used the debugger's memory viewer to see what string was being compared. Slowly, a pattern emerged. The binary wasn’t just checking a string once — it was validating chunks of it against hardcoded fragments.

---

# **Stage 4 — Mapping the Checks**

Over time, I reverse-mapped all the `strcmp` checks to reconstruct the expected input string.

Here’s how I kept track in Python (outside the binary):

```python
segments = [
    "quantum",     # from strcmp(input, "quantum")
    "mechanic",    # from strcmp(input+7, "mechanic")
    "sreveal",     # from strcmp(input+14, "sreveal")
    "truth"        # from strcmp(input+21, "truth")
]

final_input = ''.join(segments)
print(final_input)
# Output: quantummechanicsrevealtruth
```

It felt oddly poetic.

---

# **Stage 5 — Binary Patching (Optional Path)**

Out of curiosity, I decided to patch the binary to bypass the `strcmp` checks altogether. Using x32dbg’s "Assemble" feature, I turned:

```assembly
jne short loc_401234
```

into:

```assembly
nop
nop
```

Effectively short-circuiting the check.

The result? The program instantly printed the flag after startup. A nice trick, but I preferred earning the flag the clean way.

---

# **Stage 6 — Submitting the Input**

Back in the real binary, I entered:

```
quantummechanicsrevealtruth
```

The screen flickered for a second, and then the flag appeared in all its glory.

Mission complete.

---

# **Reflections — Structured Chaos**

This challenge was a classic case of symbolic dissection. There was logic behind every check, but without static analysis and dynamic confirmation, piecing the full puzzle would’ve been painful.

In the end, it wasn’t about brute-force. It was about **observation** and **strategy** — like unwrapping a riddle encoded in machine instructions.

---

# **Bonus — Scripted `strcmp` Tracing**

For later use, I saved this Python snippet to emulate the `strcmp` checkchain in future reverse engineering scenarios:

```python
def emulate_strcmp_chain(input_str):
    checks = [
        ("quantum", 0),
        ("mechanic", 7),
        ("sreveal", 14),
        ("truth", 21)
    ]
    for expected, offset in checks:
        if input_str[offset:offset+len(expected)] != expected:
            return False
    return True

print(emulate_strcmp_chain("quantummechanicsrevealtruth"))  # True
```

---

# **Epilogue — Reading the 8 Ball**

Sometimes binaries pretend to be playful. But behind the curtain is crafted logic. This Magic 8 Ball wasn’t about luck — it was about slicing through layers of obfuscation to find what lay beneath.

I closed the debugger, zipped my notes, and powered down the VM.

Reverse engineering doesn’t always give you answers.

But it always reveals intent.
