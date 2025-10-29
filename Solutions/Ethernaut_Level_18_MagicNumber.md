# Ethernaut Level 18 MagicNumber

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {
    address public solver;

    constructor() {}

    function setSolver(address _solver) public {
        solver = _solver;
    }

    /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
    */
}
```

We have to set an address in `solver`, and in this address, we have to deploy a contract that can return `42`. The most obvious thought is to deploy a contract which have a pure function that can return number 42. However, if you go compile this contract, you will see that the actual size of the bytecode is way larger than just *10 bytes*. So the most tricky part of this game is the `10 bytes` limit. In order to meet this requirement, we have to code directly in bytecode.

In order to do all these things manually in EVM bytecode, we have to first get a greater picture of the lifecycle of all contracts. When you deploy a contract, two distinct “code” pieces exist:

| **Phase**            | **Name**                   | **What It Does**                                             | **What Happens After Deployment**                            |
| -------------------- | -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **1️⃣ Creation Phase** | **Init (deployment) code** | Runs **once** when you deploy the contract. Its job is to prepare and return the *runtime* code. | Executed **once**, then **discarded forever**.               |
| **2️⃣ Runtime Phase**  | **Runtime code**           | The actual logic that stays **on-chain** and executes for every external call. | Becomes the **permanent code** stored at the contract’s address. |

So, the `10 bytes` limit is applicable for the `runtime code` that is permanently deplpoyed on the address. We will then first figure out how to code `runtime code`, then go with the `init code`.

## Runtime Code

In this phase, what we want to do is just push a value onto the stack, and then store it in memory, and then return it. This is the minimal scheme for the runtime code. We will go step by step with the stack and memory shown so that you can better understand what is going on under the hood.

Overview:

| **Bytes** | **Opcode** | **Meaning**                        |
| --------- | ---------- | ---------------------------------- |
| 60 2a     | PUSH1 0x2a | Push value 42                      |
| 60 00     | PUSH1 0x00 | Push destination offset 0          |
| 52        | MSTORE     | Store 42 into memory at position 0 |
| 60 20     | PUSH1 0x20 | Push return size (32 bytes)        |
| 60 00     | PUSH1 0x00 | Push return offset (0)             |
| f3        | RETURN     | Return 32 bytes from memory[0..31] |

### **0️⃣ Start**

- Stack: (empty)
- Memory: all zeroes (EVM clears new memory to zero)

### **1️⃣** PUSH1 0x2a

- Pushes literal 42 (decimal) onto the stack.

**Stack:**

```
0x2a
```

**Memory:** (unchanged)

### **3️⃣** **MSTORE**

- Pops:

  - offset = 0x00
  - value = 0x2a

- MSTORE takes two inpus: (offset, value).

- Stores the 32-byte representation of 0x2a at memory[0..31].

  Since EVM words are 32 bytes, it **pads left** with zeros.

**Memory [0x00..0x1f]:**

```
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 2a
```

### **4️⃣** **PUSH1 0x20**

- Pushes 32 (decimal) — how many bytes to return.

**Stack:**

```
0x20
```

### **5️⃣** **PUSH1 0x00**

- Pushes the offset 0 — where to start returning from.

**Stack (top→bottom):**

```
0x00
0x20
```

### **6️⃣** **RETURN**

- Pops:
  - offset = 0x00
  - size  = 0x20
- RETURN takes two inpus: (offset, size).
- Returns memory[0x00 .. 0x20) — i.e., 32 bytes starting at position 0.

**Returned data:**

```
000000000000000000000000000000000000000000000000000000000000002a
```

The final result of the runtime code is:

```
602a60005260206000f3
```

Exactly 10 bytes.

## Init Code

In this phase, we want to deploy the runtime code in bytecode. In actual execution order, this part gets executed first. We put this part behind so that you can better understand what these opcodes are actually doing. We will basically do a similar thing: treat runtime code as a value, push it onto the stack, store it in memory, and then return it.

Overview:

| **Bytes**               | **Opcode**                    | **Meaning**                                  |
| ----------------------- | ----------------------------- | -------------------------------------------- |
| 69 602a60005260206000f3 | PUSH10 0x602a60005260206000f3 | Push value (runtime code)                    |
| 60 00                   | PUSH1 0x00                    | Push destination offset 0                    |
| 52                      | MSTORE                        | Store runtime code into memory at position 0 |
| 60 0a                   | PUSH1 0x0a                    | Push return size (**10** bytes)              |
| 60 16                   | PUSH1 0x16                    | Push return offset (**22**)                  |
| f3                      | RETURN                        | Return **10** bytes from memory[21..31]      |

We are not going to show the full stack and memory layout since it is similar with what is shown in the runtime code part. Few things we have to pay attention to though:

	1. We have to use **PUSH10** instead of **PUSH1** if we want to push 10 bytes onto stack.
	1. The return offset have to be 0x16 (22) because the 10-byte value sits at the **right end** of the 32-byte slot. So we want to skip the first 22 bytes.

So, our final bytecode look like this:

```
69 602a60005260206000f3  60 00  52   60 0a  60 16  f3
^  ^-- 10-byte literal   ^      ^     ^      ^      ^
|                        |      |     |      |      RETURN
PUSH10                   PUSH1  MSTORE PUSH1 PUSH1
                                        len   offset
```

`MagicNumber.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {MagicNum} from "../src/MagicNumber.sol";

contract MagicNumberScript is Script {

    MagicNum magicNum = MagicNum(0xXXXXXXX);

    function run() external {
        vm.startBroadcast();
        bytes memory deploymentBytecode = hex"69602a60005260206000f3600052600a6016f3";
        address minimalContractAddress;
        assembly{
            minimalContractAddress := create(0, add(deploymentBytecode, 0x20), mload(deploymentBytecode))
        }
        magicNum.setSolver(minimalContractAddress);
        vm.stopBroadcast();
    }
}
```

Something we have to notice: we have stored the whole bytecode in a dynamic bytes memory. Then, we use inline assembly to use `create(value, memory_start, memory_size)` to get the deployment address. In this function:

- value = amount of wei to send with deployment (usually 0)
- memory_start = where in memory the init code begins
- memory_size = how many bytes to read from memory as init code
- returns the **new contract’s address** if successful, or 0 on failure.

So:

- deploymentBytecode → pointer to the *length field*
- add(deploymentBytecode, 0x20) → pointer to the *actual data*

We want the **actual bytes**, not the length, so we add 0x20. mload(ptr) reads 32 bytes of memory at ptr. Since deploymentBytecode points to the start of the array (its length slot), mload(deploymentBytecode) loads the **length of the byte array**, which is 22 in this case.

So now we have:

- memory_start = add(deploymentBytecode, 0x20) → where bytes begin
- memory_size = mload(deploymentBytecode) → how many bytes to read (22)

That’s exactly what create() needs.