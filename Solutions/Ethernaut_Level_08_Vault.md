# Ethernaut Level 8 Vault

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

In this game, we have to unlock the vault with the password. The password is stored as a `private` `bytes32` state variable. For a `private` variable or method, we cannot access it outside this contract (both external contracts and inherited contracts). However, it is `private` *only* from the perspective of other *contracts* (or, more precisely, other parts of the ABI interface). But because the data is stored on the blockchain and visible in storage, you *can* read them externally. 

```js
await web3.eth.getStorageAt(contract.address, 1)
```

We can access the storage of the contract by using the function built in `web3.js` : `web3.eth.getStorageAt(address, storage_slot)`. Solidity contracts typically have a storage structure of sequential slots of declaration order. In this case, slot 1 is where the password is stored. We can simply access the password using this command.

Then we can call `unlock()` to pass the game.

```js
await contract.unlock("0xXXXXXXXXX")
```

Note that in `web3.js` we have to add double quote so that the type can be of `bytes`.