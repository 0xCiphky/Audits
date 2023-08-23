# Beedle - Oracle Free Perpetual Lending

The audit report will be updated with the findings as soon as the CodeHawks team finalizes and releases their report.

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

  **Severity:** 
  
  High

  **Relevant GitHub Links:** 

  https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591

  **Summary:** 

  The refinance function in the Lender contract, which allows borrowers to refinance their loans, contains a double accounting error when updating the balances after a successful refinance.

  **Vulnerability Details:** 

 The refinance function erroneously updates the new lender's pool balance twice for the same loan debt.

For each refinance operation, the function validates the loan and new lender pool, calculates the new debt, updates the old and new lender's pool balances, and transfers any necessary tokens.

  ```solidity
  // update the old lenders pool
_updatePoolBalance(oldPoolId, pools[oldPoolId].poolBalance + loan.debt + lenderInterest);
pools[oldPoolId].outstandingLoans -= loan.debt;

// now lets deduct our tokens from the new pool
_updatePoolBalance(poolId, pools[poolId].poolBalance - debt);
pools[poolId].outstandingLoans += debt;
  ```
 During the refinancing process, the new lender's pool balance should be updated once to reflect the new loan debt. However, near the end of the function, the new lender's pool balance is reduced by the same loan debt again. This results in the new lender's pool balance being deducted twice for the same debt, essentially double-counting the debt.
  ```solidity
  pools[poolId].poolBalance -= debt;
  ```
  
  **Impact:** 

  Severity: High. The new lender is charged double the amount of the actual loan debt.

Likelihood: High. The refinance function is a critical part of the protocol and is likely to be used frequently.

  **Tools Used:** 

  Manual analysis

  **Recommendation:** 

  The double accounting error can be rectified by removing the second balance update for the new lender's pool balance. This should ensure that the new lender's pool balance is only reduced by the loan debt once.

</details>

---

<details>
  <summary><a id="h02---xxx"></a>[H02] - XXX</summary>
  
  <br>

  **Severity:** High

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

---

<details>
  <summary><a id="h03---xxx"></a>[H03] - XXX</summary>
  
  <br>

  **Severity:** High

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

---

<details>
  <summary><a id="h04---xxx"></a>[H04] - XXX</summary>
  
  <br>

  **Severity:** High

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

---

<details>
  <summary><a id="h05---xxx"></a>[H05] - XXX</summary>
  
  <br>

  **Severity:** High

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

---

<details>
  <summary><a id="h06---xxx"></a>[H06] - Inconsistent balance when tokens with fee on transfer are used</summary>
  
  <br>

  **Severity:** High

  **Relevant GitHub Links:** 

  https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L182

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L198

  **Summary:** 

  The Beedle contract assumes that the amount of tokens inputted matches the amount received. However, this assumption may not hold true when dealing with tokens that impose a fee on transfers. This discrepancy between the amount received and the amount accounted for could lead to a loss of funds for all parties interacting with the protocol.

  **Vulnerability Details:** 

  Let's illustrate this with an example:

Consider a scenario where a lending pool is set up with a loan token that imposes a fee on transfers.

When the addToPool function is called, the amount of tokens accounted for will be more than the actual tokens received by the protocol due to the transfer fee.

  ```solidity
  function addToPool(bytes32 poolId, uint256 amount) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (amount == 0) revert PoolConfig();
        _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
        // transfer the loan tokens from the lender to the contract
        IERC20(pools[poolId].loanToken).transferFrom(msg.sender, address(this), amount);
    }
  ```
  Later, when the removeFromPool function is called, the protocol will transfer out the full accounted amount.
  ```solidity
  function removeFromPool(bytes32 poolId, uint256 amount) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (amount == 0) revert PoolConfig();
        _updatePoolBalance(poolId, pools[poolId].poolBalance - amount);
        // transfer the loan tokens from the contract to the lender
        IERC20(pools[poolId].loanToken).transfer(msg.sender, amount);
    }
  ```
  
  **Impact:** 

  If the protocol receives fewer tokens due to a transfer fee but later sends out the full accounted amount, it will effectively lose the amount of the transfer fee. In a high-volume environment or with large-value transactions, this could lead to substantial losses over time.

  **Tools Used:** 

  Manual analysis

  **Recommendation:** 

  To mitigate this vulnerability, we recommend checking the balance before and after each transfer to accurately account for any transfer fees. This could be done by comparing the balance of the contract before and after the transferFrom call, and then updating the accounted balance based on the actual change in balance, rather than the input amount.

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
