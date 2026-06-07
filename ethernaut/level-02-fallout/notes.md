// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Fallout {
    using SafeMath for uint256;

    mapping(address => uint256) allocations;
    address payable public owner;

    /* constructor */
    function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] = allocations[msg.sender].add(msg.value);
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        allocator.transfer(allocations[allocator]);
    }

    function collectAllocations() public onlyOwner {
        msg.sender.transfer(address(this).balance);
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }}

# Ethernaut — Level 2: Fallout

**Network:** Sepolia testnet  
**Contract:** `0x779AC54394F980058ba9a040bEe7E29E109FAcac`  
**Date completed:** June 2026

---

## What the challenge was

Take ownership of the contract.  
The contract used Solidity `^0.6.0` and was supposed to set the deployer as owner inside a constructor.

---

## What confused me

Nothing blocked me this time — once I understood the constructor naming rule from reading the code, the exploit was obvious.  
The comment `/* constructor */` in the source was actually a hint that something was wrong.

---

## Vulnerability involved

**Constructor name typo (Solidity <0.8.0 constructor pattern).**

In Solidity 0.6.x, a constructor was written as a function with the **exact same name as the contract**. The contract is named `Fallout`, but the developer wrote:

```solidity
/* constructor */
function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
}
```

Notice `Fal1out` — the letter `l` was replaced with the number `1`.

Because the name didn't match the contract name `Fallout`, Solidity did **not** treat it as a constructor. It compiled as a completely normal `public` function — callable by anyone, at any time, forever.

The contract was deployed with **no owner set** (`owner` defaults to `address(0)`), and the "constructor" was just sitting there waiting to be called.

---

## How I solved it

**Step 1 — Read the source code**  
Spotted the typo: `Fal1out` vs `Fallout`.

**Step 2 — Get a level instance**  
Clicked "Get new instance" on Ethernaut and approved the MetaMask popup.

**Step 3 — Call the fake constructor**
```javascript
await contract.Fal1out()
```

**Step 4 — Verify ownership**
```javascript
await contract.owner()
// returned: 0xE7975DD97C358bD8686ee1f2a4FA054aA321d295
```

**Step 5 — Submit the instance** ✅

One transaction. Done.

---

## Key takeaway

In Solidity `<0.8.0`, if the constructor function name didn't exactly match the contract name, it became a public callable function instead — a silent, critical bug with no compiler warning.

Solidity `0.8.x` fixed this permanently by introducing the `constructor` keyword:

```solidity
constructor() public payable {
    owner = msg.sender;
}
```

No name to misspell. The bug category no longer exists in modern Solidity.

**When auditing:** always check `pragma solidity` version first. Old version pinning (`^0.6.0`) is a red flag to look harder at constructor logic.
