The `Vault` is Balancer V2's core contract. A single instance of it exists for the entire network, and it is the entity used to interact with Pools by Liquidity Providers who join and exit them, Traders who swap, and Asset Managers who withdraw and deposit tokens.

The `Vault`'s source code is split among a number of sub-contracts, with the goal of improving readability and making understanding the system easier. Most sub-contracts have been marked as `abstract` to explicitly indicate that only the full `Vault` is meant to be deployed.

For balancer v2, the swap was designed for two main core purpose;
1. Flexibility
1. Low Gas Fees


A vault contract has basically two core types of swaps which includes the Single swap and the Batch swap.

Single swap is used only when swapping 2 tokens against a single pool.While a batch swap allows for multiple pool hops.

A controller of a pool could make changes to the swap feee and other properties of the pool. This is because the pool has a more parameterized logic.

Now in balancer v2, a vault actually holds all the tokens, each pool can have its own logic indpendently of the controller and communication with the vault. 

Lets imagine you go into a vault and try to convert ETH for USDC to pool A, all the logic in this case is been handled by the pool while the vault takes care of the transaction. Basically anything can be built on top of the vault because the pool basically communicates with the vault without handling any finances but implementation of logic.

## Overview of the Vault contract
Below shows the imports associated with the vault contract, this contract takes in two interfaces of IWETH and IAuthorizer. It inherits from swaps, flashloans and the vaultAuthorization contract.

```
//Vault.sol

import "@balancer-labs/v2-interfaces/contracts/solidity-utils/misc/IWETH.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IAuthorizer.sol";

import "./VaultAuthorization.sol";
import "./FlashLoans.sol";
import "./Swaps.sol";
```

Below is the full vault contract implementation

```
//Vault.sol

contract Vault is VaultAuthorization, FlashLoans, Swaps {
    constructor(
        IAuthorizer authorizer,
        IWETH weth,
        uint256 pauseWindowDuration,
        uint256 bufferPeriodDuration
    ) VaultAuthorization(authorizer) AssetHelpers(weth) TemporarilyPausable(pauseWindowDuration, bufferPeriodDuration) {
        // solhint-disable-previous-line no-empty-blocks
    }

    function setPaused(bool paused) external override nonReentrant authenticate {
        _setPaused(paused);
    }

    // solhint-disable-next-line func-name-mixedcase
    function WETH() external view override returns (IWETH) {
        return _WETH();
    }
}
```

The vault contract also inherits from the authorization contract called VaultAuthorization, this contract serves as a mechanism to control access to permissioned functions within the Balancer V2 Vault smart contract. 

It relies on an authorizer and signature validation for access control.

The getAuthorizer function helps return the Vault's Authorizer, that is the Balancer governance contract.
```
getAuthorizer()
returns (IAuthorizer)
```

The setAuthorizer function is a function that has the functionality to set a new Authorizer for the Vault. The caller must be allowed by the current Authorizer to do this.

```
setAuthorizer(IAuthorizer newAuthorizer)

emits AuthorizerSet(IAuthorizer indexed newAuthorizer)
```


The hasApprovedRelayer function is responsible for returning true if a user has allowed relayer to act as a relayer for them.

Now what exactly are these relayers anyways, relayers are contracts that can make calls to the vault (with the transaction “sender” being any arbitrary address) and use the sender’s ERC20 vault allowance, internal balance or BPTs on their behalf.
```
hasApprovedRelayer(
    address user,
    address relayer)
returns(bool)
```

The setRelayerApproval function Grants or revokes approval for the given relayer to call Authorizer-approved functions on behalf of user.
```
setRelayerApproval(
    address sender,
    address relayer,
    bool approved)

emits RelayerApprovalChanged(address indexed relayer,
                             address indexed sender,
                             bool approved)
```

The `_authenticateFor `function within the VaultAuthorization contract is responsible for verifying the authenticity and authorization of the caller attempting to execute a function on behalf of a specified user. 

It first checks if the caller (msg.sender) is different from the specified user (user). If they are the same, it implies that the user is directly calling the function, and no further validation is required.


If the caller is not the user, it verifies if the caller is authorized by the Authorizer to act as a relayer for the specified user. 

This authorization ensures that certain functions can be executed by trusted parties on behalf of users.

```
function _authenticateFor(address user) internal {
        if (msg.sender != user) {
            // In this context, 'permission to call a function' means 'being a relayer for a function'.
            _authenticateCaller();

            // Being a relayer is not sufficient: `user` must have also approved the caller either via
            // `setRelayerApproval`, or by providing a signature appended to the calldata.
            if (!_hasApprovedRelayer(user, msg.sender)) {
                _validateExtraCalldataSignature(user, Errors.USER_DOESNT_ALLOW_RELAYER);
            }
        }
    }
```

## Flashloans

## Imports
```
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/helpers/BalancerErrors.sol";
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/openzeppelin/IERC20.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IFlashLoanRecipient.sol";

import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/ReentrancyGuard.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeERC20.sol";
```
Above are the following import that are associated with the Flashloans contract.

`BalancerErrors` contains error code definitions and error handling utilities specific to the Balancer protocol.

`IERC20.sol` is the interface that defines the standard ERC-20 token functions that the contract interacts with.

The interface includes standard functions such as 
1. Transfer 
2. Approve 
3. TransferFrom  
4. BalanceOf for interacting with ERC-20 tokens.

`IFlashLoanRecipient.sol` is the interface that defines the functions that a contract must implement to receive flash loans from the FlashLoans contract.

This interface has the `receiveFlashLoan`  function that specifies the logic for how the recipient contract handles the loaned tokens.

`ReentrancyGuard.sol`  implements a reentrancy guard to prevent reentrant attacks during contract execution.

`SafeERC20.sol` provides an ERC20 interface that will help prevent common vulnerabilities such as overflows and underflows.

These safetychecks includes functions like `safeTransfer`, `safeTransferFrom`, and `safeApprove` that wrap standard ERC-20 functions with additional safety checks.

## FlashLoan Contract
The Flashloan contract that is inherited by the Vault enables Flash Loans through the Vault contract.  

How this works is that it calls the `receiveFlashLoan` hook on the flash loan recipient contract, which implements the `IFlashLoanRecipient` interface.

```
recipient.receiveFlashLoan(tokens, amounts, feeAmounts, userData);
```
The line above shows how the `receiveFlashLoan` function is being called by the flashloan function

```
// Flashloan.sol

pragma solidity ^0.7.0;
pragma experimental ABIEncoderV2;

import "@balancer-labs/v2-interfaces/contracts/solidity-utils/helpers/BalancerErrors.sol";
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/openzeppelin/IERC20.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IFlashLoanRecipient.sol";

import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/ReentrancyGuard.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeERC20.sol";

import "./Fees.sol";

/**
 * @dev Handles Flash Loans through the Vault. Calls the `receiveFlashLoan` hook on the flash loan recipient
 * contract, which implements the `IFlashLoanRecipient` interface.
 */
abstract contract FlashLoans is Fees, ReentrancyGuard, TemporarilyPausable {
    using SafeERC20 for IERC20;

    function flashLoan(
        IFlashLoanRecipient recipient,
        IERC20[] memory tokens,
        uint256[] memory amounts,
        bytes memory userData
    ) external override nonReentrant whenNotPaused {
        InputHelpers.ensureInputLengthMatch(tokens.length, amounts.length);

        uint256[] memory feeAmounts = new uint256[](tokens.length);
        uint256[] memory preLoanBalances = new uint256[](tokens.length);

        // Used to ensure `tokens` is sorted in ascending order, which ensures token uniqueness.
        IERC20 previousToken = IERC20(0);

        for (uint256 i = 0; i < tokens.length; ++i) {
            IERC20 token = tokens[i];
            uint256 amount = amounts[i];

            _require(token > previousToken, token == IERC20(0) ? Errors.ZERO_TOKEN : Errors.UNSORTED_TOKENS);
            previousToken = token;

            preLoanBalances[i] = token.balanceOf(address(this));
            feeAmounts[i] = _calculateFlashLoanFeeAmount(amount);

            _require(preLoanBalances[i] >= amount, Errors.INSUFFICIENT_FLASH_LOAN_BALANCE);
            token.safeTransfer(address(recipient), amount);
        }

        recipient.receiveFlashLoan(tokens, amounts, feeAmounts, userData);

        for (uint256 i = 0; i < tokens.length; ++i) {
            IERC20 token = tokens[i];
            uint256 preLoanBalance = preLoanBalances[i];

            // Checking for loan repayment first (without accounting for fees) makes for simpler debugging, and results
            // in more accurate revert reasons if the flash loan protocol fee percentage is zero.
            uint256 postLoanBalance = token.balanceOf(address(this));
            _require(postLoanBalance >= preLoanBalance, Errors.INVALID_POST_LOAN_BALANCE);

            // No need for checked arithmetic since we know the loan was fully repaid.
            uint256 receivedFeeAmount = postLoanBalance - preLoanBalance;
            _require(receivedFeeAmount >= feeAmounts[i], Errors.INSUFFICIENT_FLASH_LOAN_FEE_AMOUNT);

            _payFeeAmount(token, receivedFeeAmount);
            emit FlashLoan(recipient, token, amounts[i], receivedFeeAmount);
        }
    }
}
```

## FlashLoanRecipient Contract

The `FlashLoanRecipient` contract triggers a flash loan by calling the flashLoan function of the Vault contract since the vault inherits from the flashloan contract.

The flashLoan function is inherited from the FlashLoans contract, allowing the Vault contract to offer flash loans to external contracts.


When the flashLoan function is called, the Vault contract temporarily lends a specified amount of tokens to the FlashLoanRecipient contract.

The FlashLoanRecipient contract receives the loaned tokens in its receiveFlashLoan function, which implements the IFlashLoanRecipient interface.

```
//FlashLoanRecipient

pragma solidity ^0.7.0;

import "@balancer-labs/v2-vault/contracts/interfaces/IVault.sol";
import "@balancer-labs/v2-vault/contracts/interfaces/IFlashLoanRecipient.sol";

contract FlashLoanRecipient is IFlashLoanRecipient {
    IVault private constant vault = "0xBA12222222228d8Ba445958a75a0704d566BF2C8";

    function makeFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        bytes memory userData
    ) external {
      vault.flashLoan(this, tokens, amounts, userData);
    }

    function receiveFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        uint256[] memory feeAmounts,
        bytes memory userData
    ) external override {
        require(msg.sender == vault);
        ...
    }
}
```
After executing the desired operations with the loaned tokens, the `FlashLoanRecipient` contract is responsible for repaying the borrowed tokens plus the accrued fee to the Vault contract.

Failure to repay the loan within the same transaction reverts the transaction and this helps to ensure the safety of the Vault contract's funds.

Inside the `receiveFlashLoan` function, the `FlashLoanRecipient` contract can perform arbitrage, rebalancing, liquidations, or any other logic that utilizes the borrowed tokens.

## Fees

```
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/helpers/BalancerErrors.sol";
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/openzeppelin/IERC20.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IVault.sol";

import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/ReentrancyGuard.sol";
import "@balancer-labs/v2-solidity-utils/contracts/math/FixedPoint.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeERC20.sol";

import "./ProtocolFeesCollector.sol";
import "./VaultAuthorization.sol";
```

The `BalancerErrors.sol` contains error code definitions and error handling

`IERC20.sol` interface defines the standard ERC-20 token functions that the contract interacts with like the transfer, approve and transferFrom functions.

`IVault.sol` is the interface that include functions that a vault contract must implement. It provides a way for contracts to interact with vaults in a standardized manner.

`ReentrancyGuard.sol` contract implements a reentrancy guard to prevent reentrant attacks ensuring the safety of the contract.

`FixedPoint.sol` provides fixed-point arithmetic utilities to include functions for fixed-point multiplication and division, which are useful for calculating fees and other financial operations.

`SafeERC20.sol` This contract provides safe ERC-20 token operations to prevent common vulnerabilities such as overflows and underflows.

`ProtocolFeesCollector.sol`includes functions for setting and retrieving fee percentages, as well as mechanisms for collecting fees from transactions.

`VaultAuthorization.sol` as explained earlier, this contract is responsible for handling access control for the vault, ensuring that only authorized users can perform certain actions.

```
abstract contract Fees is IVault {
    using SafeERC20 for IERC20;

    ProtocolFeesCollector private immutable _protocolFeesCollector;

    constructor() {
        _protocolFeesCollector = new ProtocolFeesCollector(IVault(this));
    }

    function getProtocolFeesCollector() public view override returns (IProtocolFeesCollector) {
        return _protocolFeesCollector;
    }

    /**
     * @dev Returns the protocol swap fee percentage.
     */
    function _getProtocolSwapFeePercentage() internal view returns (uint256) {
        return getProtocolFeesCollector().getSwapFeePercentage();
    }

    /**
     * @dev Returns the protocol fee amount to charge for a flash loan of `amount`.
     */
    function _calculateFlashLoanFeeAmount(uint256 amount) internal view returns (uint256) {
        // Fixed point multiplication introduces error: we round up, which means in certain scenarios the charged
        // percentage can be slightly higher than intended.
        uint256 percentage = getProtocolFeesCollector().getFlashLoanFeePercentage();
        return FixedPoint.mulUp(amount, percentage);
    }

    function _payFeeAmount(IERC20 token, uint256 amount) internal {
        if (amount > 0) {
            token.safeTransfer(address(getProtocolFeesCollector()), amount);
        }
    }
}
```
As seen obviously above, fees contract in the balancer v2 is declared as abstract because it contains unimplemented functions from the `IVault` interface. 

The `IVault` interface defines certain functions like the `setPaused` function that any contract acting as a vault must implement. 

Inheriting from `IVault` and declaring the Fees contract as abstract indicates that any contract inheriting from Fees must provide implementations for the functions defined in `IVault`.

## ProtocolFeeCollector Contract
This an auxiliary contract to the Vault, deployed by it during construction. It offloads some of the tasks the Vault performs to reduce its overall bytecode size.

The current values for all protocol fee percentages are stored here, and any tokens charged as protocol fees are sent to this contract, where they may be withdrawn by authorized entities.

All authorization tasks are delegated to the Vault's own authorizer.

The swap fee is charged whenever a swap occurs, as a percentage of the fee charged by the Pool. 

These are not actually charged on each individual swap: the `Vault` relies on the Pools being honest and reporting fees due when users join and exit them.

```
import "@balancer-labs/v2-interfaces/contracts/vault/IProtocolFeesCollector.sol";

import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/ReentrancyGuard.sol";
import "@balancer-labs/v2-solidity-utils/contracts/helpers/InputHelpers.sol";
import "@balancer-labs/v2-solidity-utils/contracts/helpers/Authentication.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeERC20.sol";
```

The `ProtocolFeesCollector` contract imports several external contracts and libraries to leverage their ?????    ?functionalities within its implementation. 

Here's a brief explanation of each import:

* IProtocolFeesCollector.sol: An interface required by the `ProtocolFeesCollector` contract, implementation includes setting and retrieving fee percentages and withdrawing collected fees.

* ReentrancyGuard.sol: This library is used to prevent reentrancy attacks by adding a mutex to functions susceptible to such attacks.

* InputHelpers.sol: Provides helper functions for ensuring input validity, such as checking input lengths.

* Authentication.sol: Authentication contract defines an authentication mechanism to control access to certain functions based on authorization rules. Notable functions include `_canPerform` and `_getAuthorizer`.

* SafeERC20.sol: A library that provides safe ERC20 token transfer functions to prevent common vulnerabilities such as reentrancy and overflows.

Some notable functions within the `ProtocolFeesCollector` contract include:

`withdrawCollectedFees:` Allows authorized contracts to withdraw collected fees for specified tokens to a recipient address.

```
    function withdrawCollectedFees(
        IERC20[] calldata tokens,
        uint256[] calldata amounts,
        address recipient
    ) external override nonReentrant authenticate {
        InputHelpers.ensureInputLengthMatch(tokens.length, amounts.length);

        for (uint256 i = 0; i < tokens.length; ++i) {
            IERC20 token = tokens[i];
            uint256 amount = amounts[i];
            token.safeTransfer(recipient, amount);
        }
    }
```

`setSwapFeePercentage` and `setFlashLoanFeePercentage`: Aids setting the swap and flash loan fee percentages, respectively.

```
    function setSwapFeePercentage(uint256 newSwapFeePercentage) external override authenticate {
        _require(newSwapFeePercentage <= _MAX_PROTOCOL_SWAP_FEE_PERCENTAGE, Errors.SWAP_FEE_PERCENTAGE_TOO_HIGH);
        _swapFeePercentage = newSwapFeePercentage;
        emit SwapFeePercentageChanged(newSwapFeePercentage);
    }
```
```

function setFlashLoanFeePercentage(uint256 newFlashLoanFeePercentage) external override authenticate {
    _require(
        newFlashLoanFeePercentage <= _MAX_PROTOCOL_FLASH_LOAN_FEE_PERCENTAGE,
        Errors.FLASH_LOAN_FEE_PERCENTAGE_TOO_HIGH
    );
    _flashLoanFeePercentage = newFlashLoanFeePercentage;
    emit FlashLoanFeePercentageChanged(newFlashLoanFeePercentage);
}
```


`getSwapFeePercentage` and `getFlashLoanFeePercentage`: These functions retrieve the current swap and flash loan fee percentages, respectively.

```
function getSwapFeePercentage() external view override returns (uint256) {
    return _swapFeePercentage;
}

function getFlashLoanFeePercentage() external view override returns (uint256) {
    return _flashLoanFeePercentage;
    }
```


`_canPerform` and `_getAuthorizer`: Both are internal functions that implement the access control mechanism based on the authorization rules defined in the `Authentication contract`.

```
    function _canPerform(bytes32 actionId, address account) internal view override returns (bool) {
        return _getAuthorizer().canPerform(actionId, account, address(this));
    }

    function _getAuthorizer() internal view returns (IAuthorizer) {
        return vault.getAuthorizer();
    }
```

## Swaps

The swaps contract in balancer v2 implements the Vault's high-level swap functionality.

Users can swap tokens with Pools by calling the `swap` and `batchSwap` functions. They need not trust the Pool contracts to do this: all security checks are made by the Vault.

The `swap` function executes a single swap, while `batchSwap` can perform multiple swaps in sequence.

In each individual swap, tokens of one kind are sent from the sender to the Pool (this is the 'token in'), and tokens of another kind are sent from the Pool to the recipient in exchange (this is the 'token out').

The following imports are used for the swap contract:

```
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/helpers/BalancerErrors.sol";
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/openzeppelin/IERC20.sol";

import "@balancer-labs/v2-interfaces/contracts/vault/IPoolSwapStructs.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IGeneralPool.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IMinimalSwapInfoPool.sol";

import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/EnumerableSet.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/ReentrancyGuard.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/EnumerableMap.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeCast.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeERC20.sol";
import "@balancer-labs/v2-solidity-utils/contracts/helpers/InputHelpers.sol";
import "@balancer-labs/v2-solidity-utils/contracts/math/Math.sol";

import "./PoolBalances.sol";
import "./balances/BalanceAllocation.sol";
```
The `BalancerErrors.sol` contains error code definitions and custom error handling.

`IERC20.sol` is responsible for providing an interface for the standard ERC20 token standard.

`IPoolSwapStructs.sol`: This data structure represents a request for a token swap, where `kind` indicates the swap type ('given in' or'given out') which indicates whether or not the amount sent by the pool is known. 

The pool receives `tokenIn` and sends `tokenOut`. `amount` is the number of `tokenIn` tokens the pool will take in, or the number of `tokenOut` tokens the Pool will send out, depending on the given swap `kind`.

All other fields are not strictly necessary for most swaps, but are provided to support advanced scenarios in some Pools. 

`poolId` is the ID of the Pool involved in the swap - this is useful for Pool contracts that implement more than one Pool.

```
interface IPoolSwapStructs {

    struct SwapRequest {
        IVault.SwapKind kind;
        IERC20 tokenIn;
        IERC20 tokenOut;
        uint256 amount;
        // Misc data
        bytes32 poolId;
        uint256 lastChangeBlock;
        address from;
        address to;
        bytes userData;
    }
}
```
`IGeneralPool` interface contains an `onSwap` function that returns the number of tokens the Pool will grant to the user in a 'given in' swap, or that the user will grant to the pool in a 'given out' swap.

This can often be implemented by a `view` function, since many pricing algorithms don't need to track state changes in swaps.

```
interface IGeneralPool is IBasePool {
    function onSwap(
        SwapRequest memory swapRequest,
        uint256[] memory balances,
        uint256 indexIn,
        uint256 indexOut
    ) external returns (uint256 amount);
}
```


The interface `IMinimalSwapInfoPool` is a simple view function that returns the number of tokens the Pool will grant to the user in a 'given in' swap, or that the user will grant to the pool in a 'given out' swap.

The onswap function in this interface is similar to that of the`IGeneralPool` onSwap function.

```
interface IMinimalSwapInfoPool is IBasePool {
    function onSwap(
        SwapRequest memory swapRequest,
        uint256 currentBalanceTokenIn,
        uint256 currentBalanceTokenOut
    ) external returns (uint256 amount);
}
```
`ReentrancyGuard.sol` is an imported library that is used to prevent reentrancy attacks by adding a mutex to functions susceptible to such attacks.

`SafeCast.sol` is useful for elimination of internal bugs as downcasting from uint256/int256 in Solidity does not revert on overflow. 

This can easily result in undesired exploitation or bugs, since developers usually assume that overflows raise errors. 

Note that the`SafeCast` libraries restores this intuition by reverting the transaction when such an operation overflows.

`SafeERC20` helps to provide wrappers around ERC-20 operations that throw on failure (when the token contract returns false). 

Tokens that return no value (and instead revert or throw on failure) are also supported, non-reverting calls are assumed to be successful.

`InputHelpers.sol` helps to provides helper functions for ensuring input validity, such as checking input lengths.

```
library InputHelpers {
    function ensureInputLengthMatch(uint256 a, uint256 b) internal pure {
        _require(a == b, Errors.INPUT_LENGTH_MISMATCH);
    }

    function ensureInputLengthMatch(
        uint256 a,
        uint256 b,
        uint256 c
    ) internal pure {
        _require(a == b && b == c, Errors.INPUT_LENGTH_MISMATCH);
    }

    function ensureArrayIsSorted(IERC20[] memory array) internal pure {
        address[] memory addressArray;
        // solhint-disable-next-line no-inline-assembly
        assembly {
            addressArray := array
        }
        ensureArrayIsSorted(addressArray);
    }

    function ensureArrayIsSorted(address[] memory array) internal pure {
        if (array.length < 2) {
            return;
        }

        address previous = array[0];
        for (uint256 i = 1; i < array.length; ++i) {
            address current = array[i];
            _require(previous < current, Errors.UNSORTED_ARRAY);
            previous = current;
        }
    }
}
```

`ensureInputLengthMatch`: Ensures matching of the input parameters and useful for validating input arrays or lists to ensure consistency.

`ensureArrayIsSorted`: This function checks whether an array of addresses or ERC20 tokens is sorted in ascending order. 

It is essential for functions that require sorted input, such as binary search algorithms or operations where sorted data is necessary for correctness.

The `Maths.sol` library is responsible for providing maths functions within the contract. Functions can include the add, sub, min and max functions.
```
library Math {
    // solhint-disable no-inline-assembly

    /**
     * @dev Returns the absolute value of a signed integer.
     */
    function abs(int256 a) internal pure returns (uint256 result) {
        // Equivalent to:
        // result = a > 0 ? uint256(a) : uint256(-a)
        assembly {
            let s := sar(255, a)
            result := sub(xor(a, s), s)
        }
    }

    /**
     * @dev Returns the addition of two unsigned integers of 256 bits, reverting on overflow.
     */
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        _require(c >= a, Errors.ADD_OVERFLOW);
        return c;
    }

    /**
     * @dev Returns the addition of two signed integers, reverting on overflow.
     */
    function add(int256 a, int256 b) internal pure returns (int256) {
        int256 c = a + b;
        _require((b >= 0 && c >= a) || (b < 0 && c < a), Errors.ADD_OVERFLOW);
        return c;
    }

    /**
     * @dev Returns the subtraction of two unsigned integers of 256 bits, reverting on overflow.
     */
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        _require(b <= a, Errors.SUB_OVERFLOW);
        uint256 c = a - b;
        return c;
    }

    /**
     * @dev Returns the subtraction of two signed integers, reverting on overflow.
     */
    function sub(int256 a, int256 b) internal pure returns (int256) {
        int256 c = a - b;
        _require((b >= 0 && c <= a) || (b < 0 && c > a), Errors.SUB_OVERFLOW);
        return c;
    }

    /**
     * @dev Returns the largest of two numbers of 256 bits.
     */
    function max(uint256 a, uint256 b) internal pure returns (uint256 result) {
        // Equivalent to:
        // result = (a < b) ? b : a;
        assembly {
            result := sub(a, mul(sub(a, b), lt(a, b)))
        }
    }

    /**
     * @dev Returns the smallest of two numbers of 256 bits.
     */
    function min(uint256 a, uint256 b) internal pure returns (uint256 result) {
        // Equivalent to `result = (a < b) ? a : b`
        assembly {
            result := sub(a, mul(sub(a, b), gt(a, b)))
        }
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a * b;
        _require(a == 0 || c / a == b, Errors.MUL_OVERFLOW);
        return c;
    }

    function div(
        uint256 a,
        uint256 b,
        bool roundUp
    ) internal pure returns (uint256) {
        return roundUp ? divUp(a, b) : divDown(a, b);
    }

    function divDown(uint256 a, uint256 b) internal pure returns (uint256) {
        _require(b != 0, Errors.ZERO_DIVISION);
        return a / b;
    }

    function divUp(uint256 a, uint256 b) internal pure returns (uint256 result) {
        _require(b != 0, Errors.ZERO_DIVISION);

        // Equivalent to:
        // result = a == 0 ? 0 : 1 + (a - 1) / b;
        assembly {
            result := mul(iszero(iszero(a)), add(1, div(sub(a, 1), b)))
        }
    }
}
```

## PoolBalances Contract
The pool balances consist of the following imports:

```
import "./Fees.sol";
import "./PoolTokens.sol";
import "./UserBalance.sol";
```

The contract stores the Asset Managers (by Pool and token), and implements the top level Asset Manager and Pool interfaces,such as registering and deregistering tokens, joining and exiting Pools, and informational functions like `getPool` and `getPoolTokens`, delegating to specialization-specific functions as needed.

The`joinPool` function Handles joining a pool by converting a JoinPoolRequest into a PoolBalanceChange and invoking `_joinOrExit`.
```
    function joinPool(
        bytes32 poolId,
        address sender,
        address recipient,
        JoinPoolRequest memory request
    ) external payable override whenNotPaused {
        // This function doesn't have the nonReentrant modifier: it is applied to `_joinOrExit` instead.

        // Note that `recipient` is not actually payable in the context of a join - we cast it because we handle both
        // joins and exits at once.
        _joinOrExit(PoolBalanceChangeKind.JOIN, poolId, sender, payable(recipient), _toPoolBalanceChange(request));
    }
```

`exitPool` function Handles exiting a pool by converting an ExitPoolRequest into a PoolBalanceChange and invoking `_joinOrExit`.

```
    function exitPool(
        bytes32 poolId,
        address sender,
        address payable recipient,
        ExitPoolRequest memory request
    ) external override {
        // This function doesn't have the nonReentrant modifier: it is applied to `_joinOrExit` instead.
        _joinOrExit(PoolBalanceChangeKind.EXIT, poolId, sender, recipient, _toPoolBalanceChange(request));
    }
```

`_toPoolBalanceChange`: Converts a `JoinPoolRequest` or `ExitPoolRequest` into a PoolBalanceChange without runtime cost.
```
    function _toPoolBalanceChange(JoinPoolRequest memory request)
        private
        pure
        returns (PoolBalanceChange memory change)
    {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            change := request
        }
    }
```

`_joinOrExit` Implements both joining and exiting a pool based on the kind of operation (join or exit).
```
    function _joinOrExit(
        PoolBalanceChangeKind kind,
        bytes32 poolId,
        address sender,
        address payable recipient,
        PoolBalanceChange memory change
    ) private nonReentrant withRegisteredPool(poolId) authenticateFor(sender) {
        // This function uses a large number of stack variables (poolId, sender and recipient, balances, amounts, fees,
        // etc.), which leads to 'stack too deep' issues. It relies on private functions with seemingly arbitrary
        // interfaces to work around this limitation.

        InputHelpers.ensureInputLengthMatch(change.assets.length, change.limits.length);

        IERC20[] memory tokens = _translateToIERC20(change.assets);
        bytes32[] memory balances = _validateTokensAndGetBalances(poolId, tokens);

        (
            bytes32[] memory finalBalances,
            uint256[] memory amountsInOrOut,
            uint256[] memory paidProtocolSwapFeeAmounts
        ) = _callPoolBalanceChange(kind, poolId, sender, recipient, change, balances);
        PoolSpecialization specialization = _getPoolSpecialization(poolId);
        if (specialization == PoolSpecialization.TWO_TOKEN) {
            _setTwoTokenPoolCashBalances(poolId, tokens[0], finalBalances[0], tokens[1], finalBalances[1]);
        } else if (specialization == PoolSpecialization.MINIMAL_SWAP_INFO) {
            _setMinimalSwapInfoPoolBalances(poolId, tokens, finalBalances);
        } else {
            _setGeneralPoolBalances(poolId, finalBalances);
        }

        bool positive = kind == PoolBalanceChangeKind.JOIN; // Amounts in are positive, out are negative
        emit PoolBalanceChanged(
            poolId,
            sender,
            tokens,
            _unsafeCastToInt256(amountsInOrOut, positive),
            paidProtocolSwapFeeAmounts
        );
    }
```

`_callPoolBalanceChange`: Calls the corresponding pool hook to get the amounts in/out plus protocol fee amounts, and performs associated token transfers and fee payments.
```
    function _callPoolBalanceChange(
        PoolBalanceChangeKind kind,
        bytes32 poolId,
        address sender,
        address payable recipient,
        PoolBalanceChange memory change,
        bytes32[] memory balances
    )
        private
        returns (
            bytes32[] memory finalBalances,
            uint256[] memory amountsInOrOut,
            uint256[] memory dueProtocolFeeAmounts
        )
    {
        (uint256[] memory totalBalances, uint256 lastChangeBlock) = balances.totalsAndLastChangeBlock();

        IBasePool pool = IBasePool(_getPoolAddress(poolId));
        (amountsInOrOut, dueProtocolFeeAmounts) = kind == PoolBalanceChangeKind.JOIN
            ? pool.onJoinPool(
                poolId,
                sender,
                recipient,
                totalBalances,
                lastChangeBlock,
                _getProtocolSwapFeePercentage(),
                change.userData
            )
            : pool.onExitPool(
                poolId,
                sender,
                recipient,
                totalBalances,
                lastChangeBlock,
                _getProtocolSwapFeePercentage(),
                change.userData
            );

        InputHelpers.ensureInputLengthMatch(balances.length, amountsInOrOut.length, dueProtocolFeeAmounts.length);
        finalBalances = kind == PoolBalanceChangeKind.JOIN
            ? _processJoinPoolTransfers(sender, change, balances, amountsInOrOut, dueProtocolFeeAmounts)
            : _processExitPoolTransfers(recipient, change, balances, amountsInOrOut, dueProtocolFeeAmounts);
    }
```

`_processJoinPoolTransfers`: Transfers amounts in from the sender, checks their limits, and pays accumulated protocol swap fees.
```
    function _processJoinPoolTransfers(
        address sender,
        PoolBalanceChange memory change,
        bytes32[] memory balances,
        uint256[] memory amountsIn,
        uint256[] memory dueProtocolFeeAmounts
    ) private returns (bytes32[] memory finalBalances) {
        // We need to track how much of the received ETH was used and wrapped into WETH to return any excess.
        uint256 wrappedEth = 0;

        finalBalances = new bytes32[](balances.length);
        for (uint256 i = 0; i < change.assets.length; ++i) {
            uint256 amountIn = amountsIn[i];
            _require(amountIn <= change.limits[i], Errors.JOIN_ABOVE_MAX);

            // Receive assets from the sender - possibly from Internal Balance.
            IAsset asset = change.assets[i];
            _receiveAsset(asset, amountIn, sender, change.useInternalBalance);

            if (_isETH(asset)) {
                wrappedEth = wrappedEth.add(amountIn);
            }

            uint256 feeAmount = dueProtocolFeeAmounts[i];
            _payFeeAmount(_translateToIERC20(asset), feeAmount);

            finalBalances[i] = (amountIn >= feeAmount) // This lets us skip checked arithmetic
                ? balances[i].increaseCash(amountIn - feeAmount)
                : balances[i].decreaseCash(feeAmount - amountIn);
        }

        _handleRemainingEth(wrappedEth);
    }
```

`_processExitPoolTransfers`: Transfers amounts out to the recipient, checks their limits, and pays accumulated protocol swap fees from the pool.
```
    function _processExitPoolTransfers(
        address payable recipient,
        PoolBalanceChange memory change,
        bytes32[] memory balances,
        uint256[] memory amountsOut,
        uint256[] memory dueProtocolFeeAmounts
    ) private returns (bytes32[] memory finalBalances) {
        finalBalances = new bytes32[](balances.length);
        for (uint256 i = 0; i < change.assets.length; ++i) {
            uint256 amountOut = amountsOut[i];
            _require(amountOut >= change.limits[i], Errors.EXIT_BELOW_MIN);

            // Send tokens to the recipient - possibly to Internal Balance
            IAsset asset = change.assets[i];
            _sendAsset(asset, amountOut, recipient, change.useInternalBalance);

            uint256 feeAmount = dueProtocolFeeAmounts[i];
            _payFeeAmount(_translateToIERC20(asset), feeAmount);

            // Compute the new Pool balances. A Pool's token balance always decreases after an exit (potentially by 0).
            finalBalances[i] = balances[i].decreaseCash(amountOut.add(feeAmount));
        }
    }
```

`_validateTokensAndGetBalances`: Validates tokens for a given pool against expected tokens and retrieves their balances.
```
    function _validateTokensAndGetBalances(bytes32 poolId, IERC20[] memory expectedTokens)
        private
        view
        returns (bytes32[] memory)
    {
        (IERC20[] memory actualTokens, bytes32[] memory balances) = _getPoolTokens(poolId);
        InputHelpers.ensureInputLengthMatch(actualTokens.length, expectedTokens.length);
        _require(actualTokens.length > 0, Errors.POOL_NO_TOKENS);

        for (uint256 i = 0; i < actualTokens.length; ++i) {
            _require(actualTokens[i] == expectedTokens[i], Errors.TOKENS_MISMATCH);
        }

        return balances;
    }
```

`_unsafeCastToInt256`: Casts an array of uint256 to int256, setting the sign of the result according to the positive flag.
```
    function _unsafeCastToInt256(uint256[] memory values, bool positive)
        private
        pure
        returns (int256[] memory signedValues)
    {
        signedValues = new int256[](values.length);
        for (uint256 i = 0; i < values.length; i++) {
            signedValues[i] = positive ? int256(values[i]) : -int256(values[i]);
        }
    }
```

## UserBalance Contract
The userBalance Contract implement User Balance interactions, which combine Internal Balance and using the Vault's ERC20 allowance.

Users can deposit tokens into the Vault, where they are allocated to their Internal Balance, and later transferred or withdrawn. 

It can also be used as a source of tokens when joining Pools, as a destination when exiting them, and as either when performing swaps. 

This usage of Internal Balance results in greatly reduced gas costs when compared to relying on plain ERC20 transfers, leading to large savings for frequent users.

Internal Balance management features batching, which means a single contract call can be used to perform multiple operations of different kinds, with different senders and recipients, at once.

The imports are associated with the following imports:
```
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/helpers/BalancerErrors.sol";
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/openzeppelin/IERC20.sol";

import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/ReentrancyGuard.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeCast.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeERC20.sol";
import "@balancer-labs/v2-solidity-utils/contracts/math/Math.sol";
```
All the above import functionalities are similar to that utilized in the `Swap.sol contract` stated above.

The following line `import "@balancer-labs/v2-interfaces/contracts/solidity-utils/openzeppelin/IERC20.sol";`imports the IERC20 interface from the OpenZeppelin library which provides the contract with the standard functions that an ERC-20 token contract must implement, such as transfer, approve, and transferFrom.

The contract also contains two key imports which includes,
```
import "./AssetTransfersHandler.sol";
import "./VaultAuthorization.sol";
```

`VaultAuthorization` contract has been clearly explained above.

`_depositToInternalBalance` is a function that increases the internal balance of a user for a specific token and transfers the corresponding asset to the contract, marking the sender as the recipient.

```
function _depositToInternalBalance(
    IAsset asset,
    address sender,
    address recipient,
    uint256 amount
) private {
    _increaseInternalBalance(recipient, _translateToIERC20(asset), amount);
    _receiveAsset(asset, amount, sender, false);
}
```

`_withdrawFromInternalBalance` decreases the internal balance of a user for a specific token and transfers the corresponding asset to the recipient.

It transfers the corresponding asset to the recipient while deducting the amount from the sender's internal balance.
```
function _withdrawFromInternalBalance(
    IAsset asset,
    address sender,
    address payable recipient,
    uint256 amount
) private {
    _decreaseInternalBalance(sender, _translateToIERC20(asset), amount, false);
    _sendAsset(asset, amount, recipient, false);
}
```

The function below is responsible for the transfers of tokens between the internal balances of two users.
```
function _transferInternalBalance(
    IERC20 token,
    address sender,
    address recipient,
    uint256 amount
) private {
    _decreaseInternalBalance(sender, token, amount, false);
    _increaseInternalBalance(recipient, token, amount);
}
```
Firstly it checks that the inputed amount is greater than zero then it performs the logic for transfers of tokens from the sender's internal balance to an external recipient's address and emits an event.
```
function _transferToExternalBalance(
    IERC20 token,
    address sender,
    address recipient,
    uint256 amount
) private {
    if (amount > 0) {
        token.safeTransferFrom(sender, recipient, amount);
        emit ExternalBalanceTransfer(token, sender, recipient, amount);
    }
}
```

`_validateUserBalanceOp` is simply an internal view function that validates a user balance operation, ensuring that the contract caller is allowed to perform it. 

It checks if the sender is either the contract caller or has approved the contract caller as a relayer.
```
function _validateUserBalanceOp(UserBalanceOp memory op, bool checkedCallerIsRelayer)
    private
    view
    returns (
        UserBalanceOpKind,
        IAsset,
        uint256,
        address,
        address payable,
        bool
    )
{
    address sender = op.sender;

    if (sender != msg.sender) {
        if (!checkedCallerIsRelayer) {
            _authenticateCaller();
            checkedCallerIsRelayer = true;
        }

        _require(_hasApprovedRelayer(sender, msg.sender), Errors.USER_DOESNT_ALLOW_RELAYER);
    }

    return (op.kind, op.asset, op.amount, sender, op.recipient, checkedCallerIsRelayer);
}

```

`_getInternalBalance` is an internal view function that retrieves the internal balance of a user for a specific token by return calling the `_internalTokenBalance`.
```
function _getInternalBalance(address account, IERC20 token) internal view returns (uint256) {
    return _internalTokenBalance[account][token];
}
```


`_setInternalBalance` is a private function sets the internal balance of a user for a specific token of an account to a given value and emits an event.
```
function _setInternalBalance(
    address account,
    IERC20 token,
    uint256 newBalance,
    int256 delta
) private {
    _internalTokenBalance[account][token] = newBalance;
    emit InternalBalanceChanged(account, token, delta);
}
```

`_increaseInternalBalance` function is set to override, this is essential to increase the internal balance of a user, it calls both the `_getInternalBalance` and the `_setInternalBalance` functions.
```
function _increaseInternalBalance(
    address account,
    IERC20 token,
    uint256 amount
) internal override {
    uint256 currentBalance = _getInternalBalance(account, token);
    uint256 newBalance = currentBalance.add(amount);
    _setInternalBalance(account, token, newBalance, amount.toInt256());
}
```
Similar to utilizing the `_setInternalBalance` and the `_getInternalBalance`functions in the `_increaseInternalBalance` function above, `_decreaseInternalBalance` function contains the logic to decrease the internal balance of an account. It functions by decreasing the user balance of a specific token by a given amount.
```
function _decreaseInternalBalance(
    address account,
    IERC20 token,
    uint256 amount,
    bool allowPartial
) internal override returns (uint256 deducted) {
    uint256 currentBalance = _getInternalBalance(account, token);
    _require(allowPartial || (currentBalance >= amount), Errors.INSUFFICIENT_INTERNAL_BALANCE);

    deducted = Math.min(currentBalance, amount);
    uint256 newBalance = currentBalance - deducted;
    _setInternalBalance(account, token, newBalance, -(deducted.toInt256()));
}
```

`_setInternalBalance` is a private function that sets the internal balance of a user for a specific token by calling the `_internalTokenBalance` mapping to a given value and emits an event.
```
function _setInternalBalance(
    address account,
    IERC20 token,
    uint256 newBalance,
    int256 delta
) private {
    _internalTokenBalance[account][token] = newBalance;
    emit InternalBalanceChanged(account, token, delta);
}
```
An internal view function that returns the internal balance of a user for a specific token using the `_internalTokenBalance` mapping.
```
function _getInternalBalance(address account, IERC20 token) internal view returns (uint256) {
    return _internalTokenBalance[account][token];
}
```

## AssetTransfersHandler Contract
In the balancer v2 protocol, the assetTransferHandler contract provides the necessary logic for depositing and withdrawing tokens from a Managed Pool

```
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/helpers/BalancerErrors.sol";
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/openzeppelin/IERC20.sol";
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/misc/IWETH.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IAsset.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IVault.sol";

import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/SafeERC20.sol";
import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/Address.sol";
import "@balancer-labs/v2-solidity-utils/contracts/math/Math.sol";

import "./AssetHelpers.sol";
```
`_receiveAsset` function receives `amount` of `asset` from `sender`. If `fromInternalBalance` is true, it first withdraws as much as possible from Internal Balance, then transfers any remaining amount.

If `asset` is ETH, `fromInternalBalance` must be false (as ETH cannot be held as internal balance), and the funds will be wrapped into WETH.

```
    function _receiveAsset(
        IAsset asset,
        uint256 amount,
        address sender,
        bool fromInternalBalance
    ) internal {
        if (amount == 0) {
            return;
        }

        if (_isETH(asset)) {
            _require(!fromInternalBalance, Errors.INVALID_ETH_INTERNAL_BALANCE);

            // The ETH amount to receive is deposited into the WETH contract, which will in turn mint WETH for
            // the Vault at a 1:1 ratio.

            // A check for this condition is also introduced by the compiler, but this one provides a revert reason.
            // Note we're checking for the Vault's total balance, *not* ETH sent in this transaction.
            _require(address(this).balance >= amount, Errors.INSUFFICIENT_ETH);
            _WETH().deposit{ value: amount }();
        } else {
            IERC20 token = _asIERC20(asset);

            if (fromInternalBalance) {
                // We take as many tokens from Internal Balance as possible: any remaining amounts will be transferred.
                uint256 deductedBalance = _decreaseInternalBalance(sender, token, amount, true);
                // Because `deductedBalance` will be always the lesser of the current internal balance
                // and the amount to decrease, it is safe to perform unchecked arithmetic.
                amount -= deductedBalance;
            }

            if (amount > 0) {
                token.safeTransferFrom(sender, address(this), amount);
            }
        }
    }
```

`_sendAsset` function Sends `amount` of `asset` to `recipient`. `toInternalBalance` is true, the asset is deposited as Internal Balance instead of being transferred. 

If `asset` is ETH, `toInternalBalance` must be false (as ETH cannot be held as internal balance), and the funds are instead sent directly after unwrapping WETH.
```
    function _sendAsset(
        IAsset asset,
        uint256 amount,
        address payable recipient,
        bool toInternalBalance
    ) internal {
        if (amount == 0) {
            return;
        }

        if (_isETH(asset)) {
            // Sending ETH is not as involved as receiving it: the only special behavior is it cannot be
            // deposited to Internal Balance.
            _require(!toInternalBalance, Errors.INVALID_ETH_INTERNAL_BALANCE);

            // First, the Vault withdraws deposited ETH from the WETH contract, by burning the same amount of WETH
            // from the Vault. This receipt will be handled by the Vault's `receive`.
            _WETH().withdraw(amount);

            // Then, the withdrawn ETH is sent to the recipient.
            recipient.sendValue(amount);
        } else {
            IERC20 token = _asIERC20(asset);
            if (toInternalBalance) {
                _increaseInternalBalance(recipient, token, amount);
            } else {
                token.safeTransfer(recipient, amount);
            }
        }
    }
```

In the balancer v2 protocol, the assetHandler contract provides the necessary logic for depositing and withdrawing tokens from a Managed Pool

The function _handleremainingEth handles the returns of excess ETH back to the contract caller, assuming `amountUsed` has been spent. 

Reverts if the caller sent less ETH than `amountUsed`.
```

function _handleRemainingEth(uint256 amountUsed) internal {
    _require(msg.value >= amountUsed, Errors.INSUFFICIENT_ETH);

    uint256 excess = msg.value - amountUsed;
    if (excess > 0) {
        msg.sender.sendValue(excess);
    }
    }
```
## AssetHelpers contract
This contract provides utility functions for handling different types of assets within the Balancer V2 ecosystem. 

`AssetHelpers` contract provides convenient functions for working with assets, including identifying ETH, translating assets into ERC20 token addresses, and interpreting assets as ERC20 tokens. 

This helper has the following import:
```
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/openzeppelin/IERC20.sol";
import "@balancer-labs/v2-interfaces/contracts/solidity-utils/misc/IWETH.sol";
import "@balancer-labs/v2-interfaces/contracts/vault/IAsset.sol";
```
Some notable functions associated with the AssetHelper contract include:

`_isETH(IAsset asset)`:
Determines whether an asset is ETH by comparing its address to a sentinel value (_ETH).
Returns true if the asset is ETH, false otherwise.

`_translateToIERC20(IAsset asset)`:
Translates an asset into an equivalent ERC20 token address.
If the asset represents ETH, it returns the address of the WETH contract.
Otherwise, it returns the asset address directly.

`_translateToIERC20(IAsset[] memory assets)`:
Applies the _translateToIERC20 function to each element of an array of assets.
Returns an array of equivalent ERC20 token addresses.

`_asIERC20(IAsset asset)`:
Interprets an asset as an ERC20 token.
This function should only be called on an asset if _isETH previously returned false for it, indicating that the asset is not the ETH sentinel value.
Casts the asset to an IERC20 interface.
