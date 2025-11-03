# Ethernaut Level 37 ImpersonatorTwo

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Ownable} from "openzeppelin-contracts-08/access/Ownable.sol";
import {ECDSA} from "openzeppelin-contracts-08/utils/cryptography/ECDSA.sol";
import {Strings} from "openzeppelin-contracts-08/utils/Strings.sol";

contract ImpersonatorTwo is Ownable {
    using Strings for uint256;

    error NotAdmin();
    error InvalidSignature();
    error FundsLocked();

    address public admin;
    uint256 public nonce;
    bool locked;

    constructor() payable {}

    modifier onlyAdmin() {
        require(msg.sender == admin, NotAdmin());
        _;
    }

    function setAdmin(bytes memory signature, address newAdmin) public {
        string memory message = string(abi.encodePacked("admin", nonce.toString(), newAdmin));
        require(_verify(hash_message(message), signature), InvalidSignature());
        nonce++;
        admin = newAdmin;
    }

    function switchLock(bytes memory signature) public {
        string memory message = string(abi.encodePacked("lock", nonce.toString()));
        require(_verify(hash_message(message), signature), InvalidSignature());
        nonce++;
        locked = !locked;
    }

    function withdraw() public onlyAdmin {
        require(!locked, FundsLocked());
        payable(admin).transfer(address(this).balance);
    }

    function hash_message(string memory message) public pure returns (bytes32) {
        return ECDSA.toEthSignedMessageHash(abi.encodePacked(message));
    }

    function _verify(bytes32 hash, bytes memory signature) internal view returns (bool) {
        return ECDSA.recover(hash, signature) == owner();
    }
}
```

Our goal is to drain all the balance from the contract and become an admin.

In order to become the admin and call `withdraw()`, we have to somehow call `setAdmin(bytes memory signature, address newAdmin)` once to make ourselves the admin and `switchLock(bytes memory signature)` to unlock the fund. Both functions need a signature that is signed off-chain by the owner, which seems totally impossible.

However, if we take a closer look at the contract, we know that both `setAdmin(bytes memory signature, address newAdmin)` and `switchLock(bytes memory signature)` must have been called at least once. If they have been called, there must be a trace. You can go to Etherscan to check and see the inputs of both functions. There's also a short cut. Go to the Ethernaut source code [here](https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/src/levels/ImpersonatorTwoFactory.sol), you can directly see the signatures used for these two function calls.

```solidity
bytes constant SWITCH_LOCK_SIG = abi.encodePacked(
    hex"e5648161e95dbf2bfc687b72b745269fa906031e2108118050aba59524a23c40", // r
    hex"70026fc30e4e02a15468de57155b080f405bd5b88af05412a9c3217e028537e3", // s
    uint8(27) // v
);
bytes constant SET_ADMIN_SIG = abi.encodePacked(
    hex"e5648161e95dbf2bfc687b72b745269fa906031e2108118050aba59524a23c40", // r
    hex"4c3ac03b268ae1d2aca1201e8a936adf578a8b95a49986d54de87cd0ccb68a79", // s
    uint8(27) // v
);
```

Now we can see a *great logic pitfall* here. Since both signatures share a *same* r, it means that the signer used the same random number k to generate signatures. We can directly calculate signer's private key. Now recall the elliptic cryptography we have been through in level 32 impersonator and level 35 elliptic token. Since $R=kG$ and $r=x(R) \mod n$, apparently the signer used the same k for two signatures. Now we get two equations:
$$
s_0 = k^{-1}(z_0 + re) \mod n \\
s_1 = k^{-1}(z_1 + re) \mod n
$$
We can first calculate the k:
$$
s_0 - s_1 = k^{-1}(z_0 - z_1) \mod n
$$

$$
k = Inv(s_0 - s_1) * (z_0 - z_1) \mod n
$$

Now we have
$$
e = Inv(r) * (ks_0 - z_0) \mod n
$$
We get the signer's **private key**.

Now since we also know the nonce (starts at 2 now), we can then construct new message digests and use the calculated private key to sign all of them off-chain. Then we go back and call `setAdmin(bytes memory signature, address newAdmin)` and `switchLock(bytes memory signature)` on chain to exploit this contract. We use a typescript script to help us do the job.

`recover_and_sign.ts`:

```typescript
// scripts/recover_and_sign.ts

import { ethers } from "ethers";

function modInverse(a: bigint, m: bigint): bigint {
  let [m0, x0, x1] = [m, 0n, 1n];
  let A = a % m;
  if (m === 1n) return 0n;
  while (A > 1n) {
    const q = A / m0;
    [A, m0] = [m0, A % m0];
    [x0, x1] = [x1 - q * x0, x0];
  }
  if (x1 < 0n) x1 += m;
  return x1;
}

async function main() {
  const [rHex, s0Hex, z0Hex, s1Hex, z1Hex, z2Raw, z3Raw] = process.argv.slice(2);
  console.log("------------------check input------------------");
  console.log(rHex, s0Hex, z0Hex, s1Hex, z1Hex, z2Raw, z3Raw);
  console.log("-----------------------------------------------");

  if (!rHex || !s0Hex || !z0Hex || !s1Hex || !z1Hex || !z2Raw || !z3Raw) {
    console.error("Usage: node recover_and_sign.ts <r> <s0> <z0> <s1> <z1> <z2> <z3>");
    process.exit(1);
  }

  const r = BigInt(rHex);
  const s0 = BigInt(s0Hex);
  const z0 = BigInt(z0Hex);
  const s1 = BigInt(s1Hex);
  const z1 = BigInt(z1Hex);

  const n = BigInt("0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141");

  // compute nonce k and private key x
  const k = ((z0 - z1) * modInverse(s0 - s1, n)) % n;
  const x = ((s0 * k - z0) * modInverse(r, n)) % n;

  // convert private key bigint → 32-byte hex string
  const xNorm = x < 0n ? x + n : x;
  const privKeyHex = ethers.toBeHex(xNorm, 32);

  const wallet = new ethers.Wallet(privKeyHex);

  // normalize input digests to 32-byte hex
  const z2 = ethers.toBeHex(BigInt(z2Raw), 32);
  const z3 = ethers.toBeHex(BigInt(z3Raw), 32);

  const signingKey = wallet.signingKey;
  const sig2 = signingKey.sign(z2);
  const sig3 = signingKey.sign(z3);

  // serialize to r || s || v hex
  const sig2Hex = ethers.concat([sig2.r, sig2.s, ethers.toBeHex(sig2.v, 1)]);
  const sig3Hex = ethers.concat([sig3.r, sig3.s, ethers.toBeHex(sig3.v, 1)]);

  console.log(sig2Hex);
  console.log(sig3Hex);
}

main().catch((err) => {
  console.error("Error:", err);
  process.exit(1);
});
```

`Impersonator.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Script, console} from "forge-std/Script.sol";
import {IImpersonatorTwo} from "../src/Interface.sol";

contract ImpersonatorTwoScript is Script {
    address constant instanceAddr = 0xea630140602d3551FBC00E4a6E67f8B95f9c213A;
    address constant ADMIN = 0xADa4aFfe581d1A31d7F75E1c5a3A98b2D4C40f68;

    // Signatures for SWITCH_LOCK with nonce 0
    bytes32 r0 = hex"e5648161e95dbf2bfc687b72b745269fa906031e2108118050aba59524a23c40";
    bytes32 s0 = hex"70026fc30e4e02a15468de57155b080f405bd5b88af05412a9c3217e028537e3";
    uint8 v0 = 27;
    // Signatures for SET_ADMIN with nonce 1
    bytes32 r1 = hex"e5648161e95dbf2bfc687b72b745269fa906031e2108118050aba59524a23c40";
    bytes32 s1 = hex"4c3ac03b268ae1d2aca1201e8a936adf578a8b95a49986d54de87cd0ccb68a79";
    uint8 v1 = 27;
    
    IImpersonatorTwo instance = IImpersonatorTwo(instanceAddr);


    function run() external {
        vm.startBroadcast();
        address player = msg.sender;

        bytes32 z0 = IImpersonatorTwo(instanceAddr).hash_message(string(abi.encodePacked("lock", "0")));
        bytes32 z1 = IImpersonatorTwo(instanceAddr).hash_message(string(abi.encodePacked("admin", "1", ADMIN)));

        bytes32 z2 = IImpersonatorTwo(instanceAddr).hash_message(string(abi.encodePacked("lock", "2")));
        bytes32 z3 = IImpersonatorTwo(instanceAddr).hash_message(string(abi.encodePacked("admin", "3", player)));

        // build FFI command
        string[] memory cmd = new string[](10);
        cmd[0] = "yarn";
        cmd[1] = "ts-node";                              // or your ts runner
        cmd[2] = "./recover_and_sign.ts";
        cmd[3] = vm.toString(r0); // pass r 
        cmd[4] = vm.toString(s0); // s0
        cmd[5] = vm.toString(z0);         // z0
        cmd[6] = vm.toString(s1); // s1
        cmd[7] = vm.toString(z1);         // z1
        cmd[8] = vm.toString(z2);         // z2
        cmd[9] = vm.toString(z3);         // z3

        bytes memory out = vm.ffi(cmd);
        console.log("Signature output:\n", string(out));

        vm.stopBroadcast();
    }
}
```

Note that I have turned `ffi=true` in `foundry.toml`. So we can use `yarn` to run our typescript script in our solidity script. After we have recovered the private key and signed the new signatures, we can call the functions.

`Solution.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Script, console} from "forge-std/Script.sol";
import {IImpersonatorTwo} from "../src/Interface.sol";

contract Solution is Script {
    address constant instanceAddr = 0xea630140602d3551FBC00E4a6E67f8B95f9c213A;
    IImpersonatorTwo instance = IImpersonatorTwo(instanceAddr);
    bytes sig2 = hex"84fe83b973cb36a59d8a0c7dde2d6a34e9e04a2c5b3d3a9470ea2ff338326f9a0fe56abd691abe411f69fbd82ca10fcdb13db04dc428b944f2de1bb8c801a6671b";
    bytes sig3 = hex"64ed58ff0fa4a85cbcf188cdaa13c05fb8f01a6e2ef9c278ca9025364a1d5e022f1b372cac547b6b64ddc03c0eeaf5bcecab249a129ef63a017eeac47e7b7c971b";

    function run() external {
        vm.startBroadcast();

        instance.switchLock(sig2);
        instance.setAdmin(sig3, msg.sender);
        instance.withdraw();

        console.log("New admin:", instance.admin());
        console.log("Contract balance:", address(instance).balance);

        vm.stopBroadcast();
    }
}
```

