# Ethernaut Level 26 DoubleEntryPoint

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
    function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
    mapping(address => IDetectionBot) public usersDetectionBots;
    mapping(address => uint256) public botRaisedAlerts;

    function setDetectionBot(address detectionBotAddress) external override {
        usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
    }

    function notify(address user, bytes calldata msgData) external override {
        if (address(usersDetectionBots[user]) == address(0)) return;
        try usersDetectionBots[user].handleTransaction(user, msgData) {
            return;
        } catch {}
    }

    function raiseAlert(address user) external override {
        if (address(usersDetectionBots[user]) != msg.sender) return;
        botRaisedAlerts[msg.sender] += 1;
    }
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if (forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(address to, uint256 value, address origSender)
        public
        override
        onlyDelegateFrom
        fortaNotify
        returns (bool)
    {
        _transfer(origSender, to, value);
        return true;
    }
}
```

We have to 1) deploy a detection bot and 2) register this bot to the `forta` address to protect `cryptoVault` from being swept. It is supposed that the `underlying` token, which is an instance of `DoubleEntryPoint` contract, would never be swept from the vault. We are given the address of `DoubleEntryPoint` contract instance in the beginning.

In function `sweepToken(IERC20 token)`, we can see that this condition: `require(token != underlying, "Can't transfer underlying token");` blocks users from transferring underlying tokens. However, since the underlying token is delegated from the `legacy` token, we can transfer the `legacy` token, which triggers the `delegateTransfer(address to, uint256 value, address origSender)` function. 

In order to prevent this "double entrypoint attack", we have to register a detection bot to `forta` so that it will revert the transaction in `fortaNotify`.

`DetectionBot.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

interface IDoubleEntryPoint {
    function cryptoVault() external view returns (address);
    function forta() external view returns (address);
}

contract DetectionBot is IDetectionBot {
    IForta forta;
    address public immutable vault;

    constructor (address _fortaAddress, address _vault) {
        forta = IForta(_fortaAddress);
        vault = _vault;
    }

    function handleTransaction(address user, bytes calldata msgData) external override {
        (, , address origSender) = abi.decode(msgData[4:], (address, uint256, address));
        if (origSender == vault) {
            forta.raiseAlert(user);
        }
    }
}
```

In `handleTransaction()`, we decode the `msg.data` to check whether it will transfer token out of the vault. Notice that we get rid of the first 4 bytes of data because these are the places for the function selector. 

`DoubleEntryPoint.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {IDetectionBot, IForta, IDoubleEntryPoint, DetectionBot} from "../src/DetectionBot.sol";

contract DoubleEntryPointScript is Script {
    address constant DOUBLE_ENTRY_POINT_ADDRESS = 0xXXXXXXXX;
    IDoubleEntryPoint doubleEntryPoint = IDoubleEntryPoint(DOUBLE_ENTRY_POINT_ADDRESS);

    address vaultAddress = doubleEntryPoint.cryptoVault();
    address fortaAddress = doubleEntryPoint.forta();

    IForta forta = IForta(fortaAddress);

    function run() external {
        vm.startBroadcast();
        DetectionBot bot = new DetectionBot(fortaAddress, vaultAddress);
        forta.setDetectionBot(address(bot));
        vm.stopBroadcast();
    }
}
```

