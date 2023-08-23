# Beedle - Oracle Free Perpetual Lending

---

## About

Oracle free peer to peer perpetual lending.

---

## Scope

---

## Findings Summary

| ID  | Title                            | Severity   |
|-----|----------------------------------|------------|
| [H01](#h01---xxx) | Refinance function contains a double accounting error                              | High       |
| [H02](#h02---xxx) | Failure in sellProfits Function due to absence of Token Approval                              | High       |
| [H03](#h03---xxx) | buyLoan function can be exploited to break protocol invariants                             | High       |
| [H04](#h04---xxx) | Lack of Slippage Control in sellProfits function                              | High       |
| [H05](#h05---xxx) | No Precision Scaling                             | High       |
| [H06](#h06---xxx) | Inconsistent balance when tokens with fee on transfer are used                             | High       |
| [M01](#m01---xxx) | XXX                              | Medium     |
| [L01](#l01---xxx) | XXX                              | Low/Info   |

---

## Detailed Findings

<details>
  <summary><a id="h01---xxx"></a>[H01] - Refinance function contains a double accounting error</summary>
  
  <br>

## **Severity:** 
  
- High

## **Relevant GitHub Links:** 

- https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591

## **Summary:** 

- The refinance function in the Lender contract, which allows borrowers to refinance their loans, contains a double accounting error when updating the balances after a successful refinance.

## **Vulnerability Details:** 

- The refinance function erroneously updates the new lender's pool balance twice for the same loan debt.

- For each refinance operation, the function validates the loan and new lender pool, calculates the new debt, updates the old and new lender's pool balances, and transfers any necessary tokens.

```solidity
  // update the old lenders pool
_updatePoolBalance(oldPoolId, pools[oldPoolId].poolBalance + loan.debt + lenderInterest);
pools[oldPoolId].outstandingLoans -= loan.debt;

// now lets deduct our tokens from the new pool
_updatePoolBalance(poolId, pools[poolId].poolBalance - debt);
pools[poolId].outstandingLoans += debt;
```

- During the refinancing process, the new lender's pool balance should be updated once to reflect the new loan debt. However, near the end of the function, the new lender's pool balance is reduced by the same loan debt again. This results in the new lender's pool balance being deducted twice for the same debt, essentially double-counting the debt.
  
```solidity
  pools[poolId].poolBalance -= debt;
```
  
## **Impact:** 

- Severity: High. The new lender is charged double the amount of the actual loan debt.

- Likelihood: High. The refinance function is a critical part of the protocol and is likely to be used frequently.

## **Tools Used:** 

- Manual analysis

## **Recommendation:** 

- The double accounting error can be rectified by removing the second balance update for the new lender's pool balance. This should ensure that the new lender's pool balance is only reduced by the loan debt once.

</details>

---

<details>
  <summary><a id="h02---xxx"></a>[H02] - Failure in sellProfits Function due to absence of Token Approval</summary>
  
  <br>

## **Severity:** 
  
- High

## **Relevant GitHub Links:** 

- [https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L26)

## **Summary:** 

- The sellProfits function, part of the Fees contract, is designed to swap tokens acquired from liquidations and fees for WETH. However, the function fails to approve the Uniswap v3 router to withdraw tokens from the contract. This oversight means the function will always revert, making it unusable. As noted in Uniswap's documentation, the contract must approve the router to withdraw the necessary tokens to execute the swap.

## **Vulnerability Details:** 

 - Here's the relevant part of the sellProfits function:

```solidity
  /// @notice swap loan tokens for collateral tokens from liquidations
/// @param _profits the token to swap for WETH
    function sellProfits(address _profits) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: _profits,
            tokenOut: WETH,
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amount,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        });

        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));

    }
  ```
  
## **Impact:** 

- High: The lack of approval prevents the sellProfits function from executing correctly, rendering it unusable.

## **Tools Used:** 

- Manual analysis

## **Recommendation:** 

- Implement the necessary approve call within the sellProfits function to provide the Uniswap v3 router with the necessary permissions to withdraw the required tokens.

</details>

---

<details>
  <summary><a id="h03---xxx"></a>[H03] - buyLoan function can be exploited to break protocol invariants</summary>
  
  <br>

## **Severity:** 
  
  - High

## **Relevant GitHub Links:** 

  - [https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465)

## **Summary:** 

  - The buyLoan function in the Lender contract may be manipulated to violate protocol invariants, due to a discrepancy in the assignment of the loan.lender field.

## **Vulnerability Details:** 

- The buyLoan function in the Lender contract allows lenders to acquire loans from other lenders. However, this function may be exploited due to an issue with the assignment of the new lender.

- The function takes two parameters: loanId to identify the loan to be purchased and poolId to identify the pool that will be acquiring the loan.

```solidity
 function buyLoan(uint256 loanId, bytes32 poolId) public {
  ```

- A check is performed to confirm that the pool has sufficient funds to cover the total debt of the loan. If the pool meets the requirements, its balance is updated to include the new loan.
  
```solidity
  // reject if the pool is not big enough
uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall();

// if they do have a big enough pool then transfer from their pool
_updatePoolBalance(poolId, pools[poolId].poolBalance - totalDebt);
pools[poolId].outstandingLoans += totalDebt;
  ```

- The vulnerability arises from the fact that the new lender is set to msg.sender, rather than the owner of the pool that is acquiring the loan. This can lead to discrepancies, as the pool specified in the function parameters is used to verify and update balances, while the loan itself is updated with msg.sender as the new lender.
  
```solidity
  // update the loan with the new info
loans[loanId].lender = msg.sender;
loans[loanId].interestRate = pools[poolId].interestRate;
loans[loanId].startTimestamp = block.timestamp;
loans[loanId].auctionStartTimestamp = type(uint256).max;
loans[loanId].debt = totalDebt;
  ```
Consider the following attack scenario:

- Lender 1 and Lender 2 each set up a pool.
- Borrower 1 borrows from Lender 1’s pool.
- Lender 1 initiates an auction for Borrower 1’s loan.
- A malicious actor purchases Borrower 1’s loan, specifying Lender 2's pool as the acquiring pool.
- Lender 2's pool balance is updated to account for the purchase.
- The malicious actor becomes the new lender for the loan, as they were the msg.sender.
  
## **Impact:** 

- Severity: High. The targeted lender could lose funds as a result of this exploit.

- Likelihood: High. Any actor within the protocol can execute this exploit.

## **Tools Used:** 

- Manual analysis

- Foundry

## **Recommendation:** 

This vulnerability can be addressed in two ways, depending on the design choice of the protocol:

  - Add an early check to confirm that msg.sender matches the owner of the lender pool specified in the function parameters. This would ensure that only the owner of the lender pool can call the buyLoan function.
  - Change the loan.lender assignment from msg.sender to pool.lender. This would allow any actor to call buyLoan for any validated pool, but ensure that the new lender is correctly set to the owner of the acquiring pool.

</details>

---

<details>
  <summary><a id="h04---xxx"></a>[H04] - Lack of Slippage Control in sellProfits function </summary>
  
  <br>

## **Severity:** 
  
- High

## **Relevant GitHub Links:** 

- [https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L37)

## **Summary:** 

- The sellProfits function in the Fees contract is employed to swap tokens earned from liquidations and fees to WETH. This operation is performed via the swapExactInputSingle function in the Uniswap v3 router, which exchanges a fixed quantity of one token for the maximum possible amount of another token. The issue arises due to the amountOutMinimum parameter being hardcoded to 0, which leaves the swap susceptible to front-running attacks that could result in a loss of protocol funds.

## **Vulnerability Details:** 

An attacker could potentially exploit this vulnerability in the following way:

- The attacker identifies a sellProfits transaction for a substantial amount in the mempool.
- The attacker then proceeds to sandwich the Uniswap swap, which could cause a significant loss of funds for the protocol due to the absence of slippage control.

The code snippet of the vulnerable function:

```solidity
/// @notice swap loan tokens for collateral tokens from liquidations
/// @param _profits the token to swap for WETH
    function sellProfits(address _profits) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: _profits,
            tokenOut: WETH,
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amount,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        });

        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
```
  
## **Impact:** 

- A front-running attack could potentially lead to a significant loss of protocol funds.

## **Tools Used:** 

- Manual analysis

## **Recommendation:** 

- Implement slippage control for the sellProfits function by setting a reasonable value for amountOutMinimum rather than hardcoding it to 0. This would limit the potential price impact of large swaps.

</details>

---

<details>
  <summary><a id="h05---xxx"></a>[H05] - No Precision Scaling </summary>
  
  <br>

## **Severity:** 
  
- High

## **Relevant GitHub Links:** 

- [https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L246)

## **Summary:** 

- The contracts calculations assumes that both the debt and collateral variables are represented in tokens with the same decimals.

## **Vulnerability Details:** 

lets look at an example

- The borrow function in the given contract calculates a loanRatio to determine the risk associated with a loan based on the debt and collateral provided.

```solidity
 uint256 loanRatio = (debt * 10 ** 18) / collateral;
```

- In scenarios where debt and collateral are tokens with different decimal precision, such as DAI (18 decimals) and USDC (6 decimals), the loanRatio can result in incorrect risk management as it is used to enforce the maximum loan-to-value ratio:
  
```solidity
  if (loanRatio > pool.maxLoanRatio) revert RatioTooHigh();
```
  
## **Impact:** 

- This can lead to loans with a higher actual ratio than intended, exposing lenders to higher default risks.

## **Tools Used:** 

- Manual analysis

## **Recommendation:** 

- When combining amounts of multiple tokens that may have different precision, convert all of the amounts into the same precision before any computation.

</details>

---

<details>
  <summary><a id="h06---xxx"></a>[H06] - Inconsistent balance when tokens with fee on transfer are used</summary>
  
  <br>

## **Severity:** 

- High

## **Relevant GitHub Links:** 

- https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L182

- https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L198

## **Summary:** 

- The Beedle contract assumes that the amount of tokens inputted matches the amount received. However, this assumption may not hold true when dealing with tokens that impose a fee on transfers. This discrepancy between the amount received and the amount accounted for could lead to a loss of funds for all parties interacting with the protocol.

## **Vulnerability Details:** 

Let's illustrate this with an example:

- Consider a scenario where a lending pool is set up with a loan token that imposes a fee on transfers.

- When the addToPool function is called, the amount of tokens accounted for will be more than the actual tokens received by the protocol due to the transfer fee.

```solidity
  function addToPool(bytes32 poolId, uint256 amount) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (amount == 0) revert PoolConfig();
        _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
        // transfer the loan tokens from the lender to the contract
        IERC20(pools[poolId].loanToken).transferFrom(msg.sender, address(this), amount);
    }
```

- Later, when the removeFromPool function is called, the protocol will transfer out the full accounted amount.
  
  ```solidity
  function removeFromPool(bytes32 poolId, uint256 amount) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (amount == 0) revert PoolConfig();
        _updatePoolBalance(poolId, pools[poolId].poolBalance - amount);
        // transfer the loan tokens from the contract to the lender
        IERC20(pools[poolId].loanToken).transfer(msg.sender, amount);
    }
  ```
  
## **Impact:** 

- If the protocol receives fewer tokens due to a transfer fee but later sends out the full accounted amount, it will effectively lose the amount of the transfer fee. In a high-volume environment or with large-value transactions, this could lead to substantial losses over time.

## **Tools Used:** 

- Manual analysis

## **Recommendation:** 

- To mitigate this vulnerability, we recommend checking the balance before and after each transfer to accurately account for any transfer fees. This could be done by comparing the balance of the contract before and after the transferFrom call, and then updating the accounted balance based on the actual change in balance, rather than the input amount.

</details>

---

<details>
  <summary><a id="m01---xxx"></a>[M01] - XXX</summary>
  
  <br>

  **Severity:** Medium

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

---

<details>
  <summary><a id="l01---xxx"></a>[L01] - XXX</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

---
