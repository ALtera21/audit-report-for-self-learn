---
title: Protocol Audit Report
author: ALtera21
date: July 6, 2026
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
    - [\[H-1\] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput` causes protocol to take too many tokens from users, resulting in lost fees](#h-1-incorrect-fee-calculation-in-tswappoolgetinputamountbasedonoutput-causes-protocol-to-take-too-many-tokens-from-users-resulting-in-lost-fees)
    - [\[H-2\] Lack of slippage protection in `TSwapPool::swapExactOutput` causes users to potentially receive way fewer tokens](#h-2-lack-of-slippage-protection-in-tswappoolswapexactoutput-causes-users-to-potentially-receive-way-fewer-tokens)
    - [\[H-3\] `TSwapPool::sellPoolTokens` mismatches input and output tokens causing user to receive the incorrect amount of tokens](#h-3-tswappoolsellpooltokens-mismatches-input-and-output-tokens-causing-user-to-receive-the-incorrect-amount-of-tokens)
    - [\[H-4\] `TSwapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invariant of `x * y = k`](#h-4-tswappool_swap-the-extra-tokens-given-to-users-after-every-swapcount-breaks-the-protocol-invariant-of-x--y--k)
- [Medium](#medium)
    - [\[M-1\] `TSwapPool::deposit` is missing deadline checks causing transactions to complete even after the deadline](#m-1-tswappooldeposit-is-missing-deadline-checks-causing-transactions-to-complete-even-after-the-deadline)
    - [\[M-2\] Rebase, fee-on-transfer, and ERC777 tokens break protocol invariant](#m-2-rebase-fee-on-transfer-and-erc777-tokens-break-protocol-invariant)

# Protocol Summary

A smart contract application swapping any healthy ERC20 for weth based of AMM.

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

Commit Hash: 7bd52c6a75f1115b23e0cff0fa16f7c522703fcd

## Scope

```
./src/
#-- PoolFactory.sol
#-- TSwapPool.sol
```

## Roles

- LiqudityProvider: give liquidity to the pool by calling `deposit` function and receive `LP` tokens as proof of ownership of the liquidity. had the power to withdraw the liqudity in exchange for `LP` tokens.
- swapper/users: can swap their desired token for WETH or vice versa by calling either `swapExactInput` or `swapExactOutput`, can also sell their tokens for WETH by calling `sellPoolTokens` function

# Executive Summary

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 4                      |
| Medium   | 2                      |
| Low      | 0                      |
| Info     | 0                      |
| Total    | 0                      |

# Findings

# High

### [H-1] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput` causes protocol to take too many tokens from users, resulting in lost fees 

**Description:** The `getInputAmountBasedOnOutput` function is intended to calculate the amount of tokens a user should deposit given an amount of output tokens. However, the function currently miscalculate the resulting amount. When calculating the fee, it scales the amount by 10_000 instead of 1_000.   

**Impact:** Protocol takes more fees than expected from users.

**Proof of Concept:**

Add the following test function inside `test/unit/TSwapPool.t.sol` test suite

<details>
<summary>test code</summary>

```javascript
    function testSwapExactOutputMiscalculateFees() public {

        poolToken.mint(address(pool), 100e18);
        weth.mint(address(pool), 10e18);

        uint256 poolTokenOutputAmountWithNoFee = 1e18;
        uint256 supposedWethToPaidWithNoFee = 1e17;

        uint256 poolTokenBefore = poolToken.balanceOf(user); // should have 10e18
        uint256 wethBefore = weth.balanceOf(user); // should have 10e18

        vm.startPrank(user);
        weth.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(IERC20(weth), IERC20(poolToken), poolTokenOutputAmountWithNoFee, uint64(block.timestamp));
        vm.stopPrank();

        uint256 poolTokenAfter = poolToken.balanceOf(user); // should have 10e18 + 1e18
        uint256 wethAfter = weth.balanceOf(user); // shold be 10e18 - 1e17, it should be.

        uint256 expectedFeePaid = poolTokenOutputAmountWithNoFee - pool.getOutputAmountBasedOnInput(supposedWethToPaidWithNoFee, weth.balanceOf(address(pool)), poolToken.balanceOf(address(pool)));
        uint256 actualFeePaid = wethBefore - wethAfter - supposedWethToPaidWithNoFee;

        assertGt(actualFeePaid, expectedFeePaid);

        console.log("actual fee paid", actualFeePaid);
        console.log("expected fee paid", expectedFeePaid);

        console.log(poolTokenBefore);
        console.log(wethBefore);
        console.log(poolTokenAfter);
        console.log(wethAfter);
    }
```
</details>

**Recommended Mitigation:**

```diff
    function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
        return
-           ((inputReserves * outputAmount) * 10000) /
+           ((inputReserves * outputAmount) * 1000) /
            ((outputReserves - outputAmount) * 997);
    }
```

### [H-2] Lack of slippage protection in `TSwapPool::swapExactOutput` causes users to potentially receive way fewer tokens

**Description:** The `swapExactOutput` function does not include any sort of slippage protection. This function is similar to what is done in `TSwapPool::swapExactInput`, where the function specify `minOuputAmount`, the `swapExactOutput` function should sepcify a `maxInputAmount`.

**Impact:** If market conditions change before the transaction processes, the user could get a much worse swaps

**Proof of Concept:**

Consider the following scenario:
1. The price of WETH is 1_000 USDC
2. user input a `swapExactOutput` looking for 1 WETH
   1. inputToken = USDC
   2. outputToken = WETH
   3. outputAmount = 1 WETH
   4. dealine = whatever
3. The function deos not offer `maxInputAmount`
4. As the transaction is pending in the mempool, the market changes! And the price moves HUGE -> 1 weth is now 10_000 USDC. 10x more than the user expected
5. The transaction completes, but the user sent the protocol 10_000 USDC, instead of the expected 1_000 USDC

**Recommended Mitigation:** We should include a `maxInputAmount` so the user only has to spend up to a sepcific amount, and can predict how much they will spend on the protocol (IMPORTANT* SOME COULD ARGUE THAT WE COULD JUST DON'T APPROVE TOO MANY TOKENS TO LIMIT IT);

```diff
    function swapExactOutput(
        IERC20 inputToken,
+       uint256 maxInputAmount,
.
.
.
        inputAmount = getInputAmountBasedOnOutput(
            outputAmount,
            inputReserves,
            outputReserves
        );

+       if (inputAmount > maxInputAmount) {
+           revert();    
+       }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
```

### [H-3] `TSwapPool::sellPoolTokens` mismatches input and output tokens causing user to receive the incorrect amount of tokens

**Description:** The `sellPoolTokens` function is intended to allow users to easily sell pool tokens and receive WETH in exchange. Users indicate how many pool tokens they're willing to sell in the `poolTokenAmount` parameter. However, the function currently miscalculates the swapped amount.

This is due to the fact that the `swapExactOutput` function is called, whereas the `swapExactInput` function is the one that should be called. Because users specify the exact amount of input tokens, not output.

**Impact:** Users will swap the wrong amount of tokens, which is a severe disruption of protocol functionality.

**Proof of Concept:**

Add the following test function inside `test/unit/TSwapPool.t.sol` test suite

<details>
<summary>test code</summary>

```javascript
    function testsellPoolTokensIsWrong() public {
        poolToken.mint(address(pool), 100e18);
        weth.mint(address(pool), 10e18);

        uint256 poolTokenBefore = poolToken.balanceOf(user); // should have 10e18
        uint256 wethBefore = weth.balanceOf(user); // should have 10e18

        uint256 poolTokenToSell = 1e10;

        vm.startPrank(user);
        weth.approve(address(pool), type(uint256).max);
        poolToken.approve(address(pool), type(uint256).max);
        pool.sellPoolTokens(poolTokenToSell);
        vm.stopPrank();

        vm.expectRevert();
        assertEq(poolTokenBefore, 0); // the pool token should be zero
    }
```
</details>

**Recommended Mitigation:**

Consider Changing the implementation to use `swapExactInput` instead of `swapExactOutput` Note that this would also require changing the `sellPoolTokens` function to accept a new parameter (i.e. `minWethToReceive` to be passed to `swapExactInput`)

```diff
    function sellPoolTokens(
        uint256 poolTokenAmount,
+       uint256 minWethToReceive
    ) external returns (uint256 wethAmount) {
        return
+           swapExactInput(
+               i_poolToken,
+               poolTokenAmount,
+               i_wethToken,
+               minWethToReceive,
+               uint64(block.timestamp)
+            );
-           swapExactOutput(
-               i_poolToken,
-               i_wethToken,
-               poolTokenAmount,
-               uint64(block.timestamp)
-            );
    }
```

Additionally it might be wise to add a dealine to the function, as the deadline is currently being defaulted to `block.timestamp` (MEV later)

### [H-4] `TSwapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invariant of `x * y = k`

**Description:** The protocol follows a strict invariant of `x * y = k`, Where:
- `x`: The balance of the pool token
- `y`: The balance of WETH
- `k`: the constant product of the two balances

This means that whenever the balances change in the protocol, the ratio between the two amount should remain constant, hence the `k`. However, this is broken due to the extra incentive in the `_swap` function. Meaning that over time the protocol funds will be drained.

The following block of code is responsible for the issue
```javascript
        swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000); //! @audit, this is shit
        }
```

**Impact:** A user could maliciously drain the protocol of funds by doing a lot of swaps and collecting the extra incentives given out by the protocol.

More simply put, the protocol's core invariant is broken.

**Proof of Concept:**
1. A user swaps 10 times, and collects the extra incentives of `1_000_000_000_000_000_000` tokens
2. That user continues to swap untill all the protocol funds are drained

<details>
<summary>proof of code</summary>

Place the following into `TSwapPool.t.sol`

```javascript
    function testInvariantBroken() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 outputWeth = 1e16;
        // expectedDeltaX = int256(pool.getPoolTokensToDepositBasedOnWeth(poolTokenAmountAdded));
        
        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        
        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth); // the minus 1 is there bcuz the pool is losing weth // the expected weth added is supposed to be outputWeth
        
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool));

        int256 actualDeltaY = int256(endingY) - startingY;

        assertEq(actualDeltaY, expectedDeltaY);
    }
```
</details>

**Recommended Mitigation:** Remove the extra incentives. If you want to keep this in, we should account for the change in the x * y = k protocol invariant. Or, we should set aside tokens in the same way we do with fees.

```diff
-        swap_count++;
-        if (swap_count >= SWAP_COUNT_MAX) {
-           swap_count = 0;
-           outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000); //! @audit, this is shit
-        }
```

# Medium

### [M-1] `TSwapPool::deposit` is missing deadline checks causing transactions to complete even after the deadline

**Description:** The `deposit` function accepts a dealine parameter, which according to the documentation is "The deadline for the transactions to be completed by". However, this parameter is never used. AS a consequence, operations that add liquidity to the pool might be executed at unexpected times, i market conditions where the deposit rate is unfavorable. 

**Impact:** Transactions could be sent when market conditions are unfavorable to deposit, even whenr adding a deadline parameter.

**Proof of Concept:** The `deadline` parameter is unused.

**Recommended Mitigation:** Consider making the following change to the function.

```diff
    function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline //! @audit param not used
    )
        external
+       revertIfDeadlinePassed(deadline)        
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {

        ...

    }
```

### [M-2] Rebase, fee-on-transfer, and ERC777 tokens break protocol invariant
