# Ethernaut Level 12 Privacy

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {
    bool public locked = true;
    uint256 public ID = block.timestamp;
    uint8 private flattening = 10;
    uint8 private denomination = 255;
    uint16 private awkwardness = uint16(block.timestamp);
    bytes32[3] private data;

    constructor(bytes32[3] memory _data) {
        data = _data;
    }

    function unlock(bytes16 _key) public {
        require(_key == bytes16(data[2]));
        locked = false;
    }

    /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
    */
}
```

Nothing in Ethereum blockchain is really *private*. We can easily access the storage of the contract using `web3.eth.getStorageAt(address, slot_index)`. In this case, we need to `1. Find out the index of data[2]`, `2. Typecasting data[2] from bytes32 to bytes16`, `3. call unlock to modify locked`. 

Note that the storage layout of a common contract is in sequential slots in declaration order. No matter of which data type, these variables will be stored as `bytes` in each slot of size `32`. The first state variable is `bool public locked`, and it is stored as a `bytes32` in slot 0. The second state variable is `uint256 public ID`, which also takes a whole `bytes32` space, thus in slot 1. The next 3 variables, `uint8 private flattening`, `uint8 private denomination`, `uint16 private awkwardness` takes **one** slot as a whole because `8 + 8 + 16 = 32 bits`. These 3 variables are in slot 2, taking a space of 8 bytes.  So, `data[0]`, `data[1]`, `data[2]` takes slot 3, 4, 5 correspondently. 

```js
const hex32 = await web3.eth.getStorageAt(contract.address, 5)
```

`hex32` is the `data[2]` in type `bytes32`. Next, we want to typecasting it to `bytes16`, which means we only want first 16 bytes of `hex32` to be wired to the function.

```js
const key16 = "0x" + hex32.slice(2, 2 + 32)
```

Then we use `key16` to unlock:

```js
await contract.unlock(key16)
```

