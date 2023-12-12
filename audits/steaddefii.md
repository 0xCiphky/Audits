# Steadefi

---

## About

Steadefi is the next-gen DeFi protocol designed to provide the highest and most sustainable real yields to our investors without the stress of constant position management or the prolonged downturns of the crypto markets.

---

## Scope

---

## Findings Summary

| ID  | Title                            | Severity   |
|-----|----------------------------------|------------|
| [H01](#h01---xxx) | processDepositCancellation can be maliciously reverted when sending native tokens to a user                              | High       |
| [H02](#h02---xxx) | Incorrect Execution Fee Refund address on Failed Deposits or withdrawals in Strategy Vaults                              | High       |
| [M01](#m01---xxx) | emergencyPause/emergencyResume functions don’t let keeper adjust the slippage                              | Medium     |
| [M02](#m02---xxx) | emergencyPause function doesn’t account for any two-step actions currently in progress                              | Medium     |
| [M03](#m03---xxx) | emergencyResume does not handle the afterDepositCancellation case correctly                              | Medium     |
| [M04](#m04---xxx) | ExchangeRouter and gmxOracle address can’t be modified                              | Medium     |
| [M05](#m05---xxx) | Inaccurate Fee Due to missing lastFeeCollected Update Before feePerSecond Modification                              | Medium     |


---

## Detailed Findings

<details>
  <summary><a id="h01---xxx"></a>[H01] - processDepositCancellation can be maliciously reverted when sending native tokens to a user</summary>
  
  <br>

**Severity:** High

**Summary:** 

  The Strategy Vaults within the protocol use a two-step process for handling asset transfers via GMXv2. A **`createDeposit()`** transaction is followed by a callback function (**`afterDepositExecution()`** or **`afterDepositCancellation()`**) based on the transaction's success. In the event of a failed deposit due to GMX checks, a malicious user can halt the protocol by causing an intentional revert in the processDepositCancellation function.

**Vulnerability Details:** 

  The **`processDepositCancellation`** function is invoked when a deposit to the GMX fails and the corresponding **`afterDepositCancellation()`** callback is triggered in the vault's callback contract. The function is designed to refund the user's deposited assets. However, there's a vulnerability when returning native tokens through a low-level call.

```solidity
function processDepositCancellation(GMXTypes.Store storage self) external {
        GMXChecks.beforeProcessDepositCancellationChecks(self);

        // Repay borrowed assets
        GMXManager.repay(
            self, self.depositCache.borrowParams.borrowTokenAAmt, self.depositCache.borrowParams.borrowTokenBAmt
        );

        // Return user's deposited asset
        // If native token is being withdrawn, we convert wrapped to native
        if (self.depositCache.depositParams.token == address(self.WNT)) {
            self.WNT.withdraw(self.WNT.balanceOf(address(this)));
            (bool success,) = self.depositCache.user.call{value: address(this).balance}("");
            require(success, "Transfer failed.");
        } else {
            // Transfer requested withdraw asset to user
            IERC20(self.depositCache.depositParams.token).safeTransfer(
                self.depositCache.user, self.depositCache.depositParams.amt
            );
        }

        self.status = GMXTypes.Status.Open;

        emit DepositCancelled(self.depositCache.user);
    }
```

The vulnerability lies in the use of a low-level call to transfer native tokens, which checks for a successful transfer before completing the transaction. A malicious user can create a smart contract with a receive function that purposely fails, preventing the completion of the **`processDepositCancellation`** function.

**Impact:** 

The exploit can lead to the **`processDepositCancellation`** function consistently failing, which traps the contract in a perpetual "Deposit" state. This persistent state prevents any future interactions with the vault, effectively freezing its operations and could be leveraged to perform a denial-of-service attack on the protocol.

**Tools Used:** 

Manual analysis

**Recommendation:** 

To mitigate the risk, the protocol should avoid relying on the success status of the low-level call within the **`processDepositCancellation`** function. One possible solution could be implementing a try-catch mechanism around the low-level call or not requiring the success of the call for the function to proceed. Here's the updated code suggestion:

```solidity
function processDepositCancellation(GMXTypes.Store storage self) external {
        GMXChecks.beforeProcessDepositCancellationChecks(self);

		...

        // Return user's deposited asset
        // If native token is being withdrawn, we convert wrapped to native
        if (self.depositCache.depositParams.token == address(self.WNT)) {
            self.WNT.withdraw(self.WNT.balanceOf(address(this)));
            (bool success,) = self.depositCache.user.call{value: address(this).balance}("");
        } else {
	      
		...
    }
```

</details>

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
  <summary><a id="m01---xxx"></a>[M01] - XXX</summary>
  
  <br>

  **Severity:** Medium

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="m02---xxx"></a>[M02] - XXX</summary>
  
  <br>

  **Severity:** Medium

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="m03---xxx"></a>[M03] - XXX</summary>
  
  <br>

  **Severity:** Medium

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="m04---xxx"></a>[M04] - XXX</summary>
  
  <br>

  **Severity:** Medium

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="m05---xxx"></a>[M05] - XXX</summary>
  
  <br>

  **Severity:** Medium

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

---
