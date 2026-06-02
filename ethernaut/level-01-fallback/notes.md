# Ethernaut — Level 1: Fallback

**Network:** Sepolia testnet  
**Contract:** `0x1691E8EEc2a19b4A1fA0ede370519ec917D454AB`  
**Date completed:** June 2026

---

## What the challenge was

Take ownership of the contract and drain all its ETH balance.  
The contract had an owner who was the only one allowed to call `withdraw()`.

---

## What confused me

- I didn't have Sepolia ETH at first — got it free from the Google Cloud faucet (no real money needed, it's a testnet)
- I kept cancelling MetaMask popups by accident, which caused `User denied transaction signature` errors
- I tried calling `withdraw()` before actually becoming the owner, which reverted with `caller is not the owner`
- I didn't realise that `contribute()` alone wasn't enough — I also had to send ETH directly to the contract

---

## Vulnerability involved

**Insecure `receive()` fallback function.**

The contract had a `receive()` function that transferred ownership to anyone who:
1. Already had a contribution on record (`contributions[msg.sender] > 0`)
2. Sent ETH directly to the contract (not via `contribute()`)

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

Ownership logic should never live in a fallback/receive function. It's too easy to trigger accidentally or maliciously.

---

## How I solved it

**Step 1 — Fund the wallet**
On the Ethernaut site (ethernaut.openzeppelin.com), selected Sepolia as the network. 
Made sure MetaMask was also set to Sepolia to match.
Got free Sepolia ETH from: https://cloud.google.com/application/web3/faucet/ethereum/sepolia

**Step 2 — Get a level instance**  
Clicked "Get new instance" on the Ethernaut page and approved the MetaMask popup.

**Step 3 — Make a small contribution** (to pass the `contributions[msg.sender] > 0` check)
```javascript
await contract.contribute({value: toWei('0.0001')})
```

**Step 4 — Send ETH directly to trigger the `receive()` function**
```javascript
await contract.sendTransaction({value: toWei('0.0001')})
```

**Step 5 — Verify ownership transferred to my address**
```javascript
await contract.owner()
// returned: 0xE7975DD97C358bD8686ee1f2a4FA054aA321d295
```

**Step 6 — Drain the contract**
```javascript
await contract.withdraw()
```

**Step 7 — Submitted the instance** on the Ethernaut page ✅

<img width="629" height="408" alt="image" src="https://github.com/user-attachments/assets/72658896-7449-48c9-9f66-c5b2f9ef6aa9" />


---

## Key takeaway

Never put ownership transfer logic inside a `receive()` or `fallback()` function.  
These execute on plain ETH transfers and are easy to exploit if they contain privileged logic.
