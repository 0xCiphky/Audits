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
  <summary><a id="h02---xxx"></a>[H02] - Incorrect Execution Fee Refund address on Failed Deposits or withdrawals in Strategy Vaults </summary>
  
  <br>

**Severity:** High

**Summary:** 

The Strategy Vaults within the protocol use a two-step process for handling deposits/withdrawals via GMXv2. A **`createDeposit()`** transaction is followed by a callback function (**`afterDepositExecution()`** or **`afterDepositCancellation()`**) based on the transaction's success. In the event of a failed deposit due to vault health checks, the execution fee refund is mistakenly sent to the depositor instead of the keeper who triggers the deposit failure process.

**Vulnerability Details:** 

The protocol handles the deposit through the **`deposit`** function, which uses several parameters including an execution fee that refunds any excess fees. 

```solidity
function deposit(GMXTypes.DepositParams memory dp) external payable nonReentrant {
        GMXDeposit.deposit(_store, dp, false);
    }

struct DepositParams {
    // Address of token depositing; can be tokenA, tokenB or lpToken
    address token;
    // Amount of token to deposit in token decimals
    uint256 amt;
    // Minimum amount of shares to receive in 1e18
    uint256 minSharesAmt;
    // Slippage tolerance for adding liquidity; e.g. 3 = 0.03%
    uint256 slippage;
    // Execution fee sent to GMX for adding liquidity
    uint256 executionFee;
  }
```

The refund is intended for the message sender (**`msg.sender`**), which in the initial deposit case, is the depositor. This is established in the **`GMXDeposit.deposit`** function, where **`self.refundee`** is assigned to **`msg.sender`**.

```solidity
function deposit(GMXTypes.Store storage self, GMXTypes.DepositParams memory dp, bool isNative) external {
        // Sweep any tokenA/B in vault to the temporary trove for future compouding and to prevent
        // it from being considered as part of depositor's assets
        if (self.tokenA.balanceOf(address(this)) > 0) {
            self.tokenA.safeTransfer(self.trove, self.tokenA.balanceOf(address(this)));
        }
        if (self.tokenB.balanceOf(address(this)) > 0) {
            self.tokenB.safeTransfer(self.trove, self.tokenB.balanceOf(address(this)));
        }

        self.refundee = payable(msg.sender);

        ...

        _dc.depositKey = GMXManager.addLiquidity(self, _alp);

        self.depositCache = _dc;

        emit DepositCreated(_dc.user, _dc.depositParams.token, _dc.depositParams.amt);
    }
```

If the deposit passes the GMX checks, the **`afterDepositExecution`** callback is triggered, leading to **`vault.processDeposit()`** to check the vault's health. A failure here updates the status to **`GMXTypes.Status.Deposit_Failed`**. The reversal process is then handled by the **`processDepositFailure`** function, which can only be called by keepers. They pay for the transaction's gas costs, including the execution fee.

```solidity
function processDepositFailure(uint256 slippage, uint256 executionFee) external payable onlyKeeper {
        GMXDeposit.processDepositFailure(_store, slippage, executionFee);
    }
```

In **`GMXDeposit.processDepositFailure`**, **`self.refundee`** is not updated, resulting in any excess execution fees being incorrectly sent to the initial depositor, although the keeper paid for it.

```solidity
function processDepositFailure(GMXTypes.Store storage self, uint256 slippage, uint256 executionFee) external {
        GMXChecks.beforeProcessAfterDepositFailureChecks(self);

        GMXTypes.RemoveLiquidityParams memory _rlp;

        // If current LP amount is somehow less or equal to amount before, we do not remove any liquidity
        if (GMXReader.lpAmt(self) <= self.depositCache.healthParams.lpAmtBefore) {
            processDepositFailureLiquidityWithdrawal(self);
        } else {
            // Remove only the newly added LP amount
            _rlp.lpAmt = GMXReader.lpAmt(self) - self.depositCache.healthParams.lpAmtBefore;

            // If delta strategy is Long, remove all in tokenB to make it more
            // efficent to repay tokenB debt as Long strategy only borrows tokenB
            if (self.delta == GMXTypes.Delta.Long) {
                address[] memory _tokenASwapPath = new address[](1);
                _tokenASwapPath[0] = address(self.lpToken);
                _rlp.tokenASwapPath = _tokenASwapPath;

                (_rlp.minTokenAAmt, _rlp.minTokenBAmt) = GMXManager.calcMinTokensSlippageAmt(
                    self, _rlp.lpAmt, address(self.tokenB), address(self.tokenB), slippage
                );
            } else {
                (_rlp.minTokenAAmt, _rlp.minTokenBAmt) = GMXManager.calcMinTokensSlippageAmt(
                    self, _rlp.lpAmt, address(self.tokenA), address(self.tokenB), slippage
                );
            }

            _rlp.executionFee = executionFee;

            // Remove liqudity
            self.depositCache.withdrawKey = GMXManager.removeLiquidity(self, _rlp);
        }
```

The same issue occurs in the **`processWithdrawFailure`** function where the excess fees will be sent to the initial user who called withdraw instead of the keeper.

**Impact:** 

This flaw causes a loss of funds for the keepers, negatively impacting the vaults. Users also inadvertently receive extra fees that are rightfully owed to the keepers

**Tools Used:** 

manual analysis

**Recommendation:** 

The **`processDepositFailure`**  and **`processWithdrawFailure`** functions must be modified to update **`self.refundee`** to the current executor of the function, which, in the case of deposit or withdraw failure, is the keeper.

```solidity
function processDepositFailure(GMXTypes.Store storage self, uint256 slippage, uint256 executionFee) external {
        GMXChecks.beforeProcessAfterDepositFailureChecks(self);

        GMXTypes.RemoveLiquidityParams memory _rlp;

	self.refundee = payable(msg.sender);

	...
        }
```

```solidity
function processWithdrawFailure(
    GMXTypes.Store storage self,
    uint256 slippage,
    uint256 executionFee
  ) external {
    GMXChecks.beforeProcessAfterWithdrawFailureChecks(self);

    self.refundee = payable(msg.sender);

    ...
  }
```

</details>

---

<details>
  <summary><a id="m01---xxx"></a>[M01] - emergencyPause/emergencyResume functions don’t let keeper adjust the slippage</summary>
  
  <br>

**Severity:** Medium

**Summary:** 

The protocol implements an emergencyPause function to be called by approved Keepers in an emergency situation. This function is designed to convert all liquidity pool tokens back to the underlying assets and hold them in the vault, also pausing all vault activities, including asset deposits, borrows, or rebalancing. However, the function fails to allow keepers to adjust the slippage that will be used when converting all liquidity pool tokens back to the underlying assets opening the door for MEV attacks.

**Vulnerability Details:** 

The emergencyPause function is called by keepers in an emergency situation, this will call GMXManager.removeLiquidity which withdraws the protocol's liquidity from the pool without allowing for slippage adjustment

```solidity
// contract GMXVault
function emergencyPause() external payable onlyKeeper {
        GMXEmergency.emergencyPause(_store);
    }

// library GMXEmergency
function emergencyPause(
    GMXTypes.Store storage self
  ) external {
    self.refundee = payable(msg.sender);

    GMXTypes.RemoveLiquidityParams memory _rlp;

    // Remove all of the vault's LP tokens
    _rlp.lpAmt = self.lpToken.balanceOf(address(this));
    _rlp.executionFee = msg.value;

    GMXManager.removeLiquidity(
      self,
      _rlp
    );

    self.status = GMXTypes.Status.Paused;

    emit EmergencyPause();
  }
```

When liquidity is added in GMXManager.addLiquidity the minMarketTokens parameter will be zero (default value). 

When liquidity is removed in GMXManager.removeLiquidity the minLongTokenAmount and minShortTokenAmount will also be set to zero (default value).

**Impact:** 

A malicious actor can sandwich the **`emergencyPause`** function, leading to significant losses for the protocol.

Similarly, the **`emergencyResume`** function is also susceptible to this issue when it attempts to add liquidity back into the pool without slippage adjustments.

**Tools Used:** 

Manual analysis

**Recommendation:** 

The **`emergencyPause`** and **`emergencyResume`** functions should be modified to include a slippage parameter that Keepers can adjust during each call.

```solidity
function emergencyPause(uint256 slippage) external payable onlyKeeper {
        GMXEmergency.emergencyPause(_store, slippage);
    }
```

```solidity
function emergencyResume(uint256 slippage) external payable onlyOwner {
        GMXEmergency.emergencyResume(_store, slippage);
    }
```

</details>

<details>
  <summary><a id="m02---xxx"></a>[M02] - emergencyPause function doesn’t account for any two-step actions currently in progress </summary>
  
  <br>

**Severity:** Medium

**Summary:** 

  The protocol implements an emergencyPause function to be called by approved Keepers in an emergency situation. This function is designed to convert all liquidity pool tokens back to the underlying assets and hold them in the vault, also pausing all vault activities, including asset deposits, borrows, or rebalancing. However, the function fails to consider ongoing process such as deposits, withdrawals, compounding, or rebalancing, which could result in incomplete actions and potential loss of funds.

**Vulnerability Details:** 

The **`emergencyPause`** function changes the vault's status to "Paused", preventing any  asset deposits, borrows, or rebalancing. However, if the function is executed while an ongoing process is underway, it could prevent the completion of that transaction. This is due to the fact that in-progress transactions rely on the vault's status for their completion logic to execute successfully.

```solidity
function emergencyPause(
    GMXTypes.Store storage self
  ) external {
    self.refundee = payable(msg.sender);

    GMXTypes.RemoveLiquidityParams memory _rlp;

    // Remove all of the vault's LP tokens
    _rlp.lpAmt = self.lpToken.balanceOf(address(this));
    _rlp.executionFee = msg.value;

    GMXManager.removeLiquidity(
      self,
      _rlp
    );

    self.status = GMXTypes.Status.Paused;

    emit EmergencyPause();
  }
```

Assuming an **`afterDepositExecution`** callback should trigger, its completion would be blocked by the paused status set by an **`emergencyPause`** call. This causes the callback to take no further action, preventing users from receiving any vault shares they should be entitled to post-deposit.

```solidity
function afterDepositExecution(
    bytes32 depositKey,
    IDeposit.Props memory /* depositProps */,
    IEvent.Props memory /* eventData */
  ) external onlyController {
    GMXTypes.Store memory _store = vault.store();

    if (
      _store.status == GMXTypes.Status.Deposit &&
      _store.depositCache.depositKey == depositKey
    ) {
      vault.processDeposit();
    } else if (
	...
  }
```

**Impact:** 

Any in-progress transaction is not accounted for when the emergency pause is activated resulting in incomplete processes and a disruption of the protocol's accounting or a loss of funds. Furthermore the execution fee refund will be sent back to the keeper that called emergencyPause instead of the user.

**Tools Used:** 

Manual analysis

**Recommendation:** 

Implement additional functionality within the **`emergencyPause`** function to ensure that ongoing transactions are accounted for before and the protocols accounting is correctly updated.

</details>

<details>
  <summary><a id="m03---xxx"></a>[M03] - emergencyResume does not handle the afterDepositCancellation case correctly</summary>
  
  <br>

**Severity:** Medium

**Summary:** 

  The **`emergencyResume`** function is intended to recover the vault's liquidity following an **`emergencyPause`**. It operates under the assumption of a successful deposit call. However, if the deposit call is cancelled by GMX, the **`emergencyResume`** function does not account for this scenario, potentially locking funds.

**Vulnerability Details:** 

When **`emergencyResume`** is invoked, it sets the vault's status to "Resume" and deposits all LP tokens back into the pool. The function is designed to execute when the vault status is "Paused" and can be triggered by an approved keeper.

```solidity
function emergencyResume(
    GMXTypes.Store storage self
  ) external {
    GMXChecks.beforeEmergencyResumeChecks(self);

    self.status = GMXTypes.Status.Resume;

    self.refundee = payable(msg.sender);

    GMXTypes.AddLiquidityParams memory _alp;

    _alp.tokenAAmt = self.tokenA.balanceOf(address(this));
    _alp.tokenBAmt = self.tokenB.balanceOf(address(this));
    _alp.executionFee = msg.value;

    GMXManager.addLiquidity(
      self,
      _alp
    );
  }
```

Should the deposit fail, the callback contract's **`afterDepositCancellation`** is expected to revert, which does not impact the continuation of the GMX execution. After the cancellation occurs, the vault status is "Resume", and the liquidity is not re-added to the pool.

```solidity
function afterDepositCancellation(
    bytes32 depositKey,
    IDeposit.Props memory /* depositProps */,
    IEvent.Props memory /* eventData */
  ) external onlyController {
    GMXTypes.Store memory _store = vault.store();

    if (_store.status == GMXTypes.Status.Deposit) {
      if (_store.depositCache.depositKey == depositKey)
        vault.processDepositCancellation();
    } else if (_store.status == GMXTypes.Status.Rebalance_Add) {
      if (_store.rebalanceCache.depositKey == depositKey)
        vault.processRebalanceAddCancellation();
    } else if (_store.status == GMXTypes.Status.Compound) {
      if (_store.compoundCache.depositKey == depositKey)
        vault.processCompoundCancellation();
    } else {
      revert Errors.DepositCancellationCallback();
    }
  }
```

Given this, another attempt to execute **`emergencyResume`** will fail because the vault status is not "Paused".

```solidity
function beforeEmergencyResumeChecks (
    GMXTypes.Store storage self
  ) external view {
    if (self.status != GMXTypes.Status.Paused)
      revert Errors.NotAllowedInCurrentVaultStatus();
  }
```

In this state, an attempt to revert to "Paused" status via **`emergencyPause`** could fail in GMXManager.removeLiquidity, as there are no tokens to send back to the GMX pool, leading to a potential fund lock within the contract.

**Impact:** 

The current implementation may result in funds being irretrievably locked within the contract. 

**Tools Used:** 

Manual Analysis

**Recommendation:** 

To address this issue, handle the afterDepositCancellation case correctly by allowing emergencyResume to be called again.

</details>

<details>
  <summary><a id="m04---xxx"></a>[M04] - ExchangeRouter and gmxOracle address can’t be modified</summary>
  
  <br>

**Severity:** Medium

**Summary:** 

  GMXVault's current implementation sets the **`gmxOracle`** and **`exchangeRouter`** addresses at deployment with no capability to update them. Given that GMX documentation suggests the potential for these addresses to change in the future, the lack of an update mechanism could result in operational issues if and when an update is required.

”If using contracts such as the ExchangeRouter, Oracle or Reader do note that their addresses will change as new logic is added”

The GMXVault contract is initially configured with the **`gmxOracle`** and **`exchangeRouter`** addresses, during the construction of the contract. However there is no functionality to change these addresses down the line.

```solidity
constructor(string memory name, string memory symbol, GMXTypes.Store memory store_)
        ERC20(name, symbol)
        Ownable(msg.sender)
    {
			
	_store.gmxOracle = IGMXOracle(store_.gmxOracle);

        _store.exchangeRouter = IExchangeRouter(store_.exchangeRouter);
        _store.router = store_.router;
        _store.depositVault = store_.depositVault;
        _store.withdrawalVault = store_.withdrawalVault;
        _store.roleStore = store_.roleStore;

        _store.swapRouter = ISwap(store_.swapRouter);

        ...
    }
```

**Impact:** 

The inability to update these addresses means that GMXVault risks becoming incompatible with newer versions of related contracts or could continue to rely on outdated or potentially insecure versions.

**Tools Used:** 

Manual analysis

**Recommendation:** 

Add owner-only functions that enable the updating of the **`gmxOracle`** and **`exchangeRouter`** addresses.

</details>

<details>
  <summary><a id="m05---xxx"></a>[M05] - Inaccurate Fee Due to missing lastFeeCollected Update Before feePerSecond Modification</summary>
  
  <br>

**Severity:** Medium

**Summary:** 

  The protocol charges a management fee based on the **`feePerSecond`** variable, which dictates the rate at which new vault tokens are minted as fees via the **`mintFee`** function. An administrative function **`updateFeePerSecond`** allows the owner to alter this fee rate. However, the current implementation does not account for accrued fees before the update, potentially leading to incorrect fee calculation.

**Vulnerability Details:** 

The contract's logic fails to account for outstanding fees at the old rate prior to updating the **`feePerSecond`**. As it stands, the **`updateFeePerSecond`** function changes the fee rate without triggering a **`mintFee`**, which would update the **`lastFeeCollected`** timestamp and mint the correct amount of fees owed up until that point.

```solidity
function updateFeePerSecond(uint256 feePerSecond) external onlyOwner {
        _store.feePerSecond = feePerSecond;
        emit FeePerSecondUpdated(feePerSecond);
    }
```

**Scenario Illustration:**

- User A deposits, triggering **`mintFee`** and setting **`lastFeeCollected`** to the current **`block.timestamp`**.
- After two hours without transactions, no additional **`mintFee`** calls occur.
- The owner invokes **`updateFeePerSecond`** to increase the fee by 10%.
- User B deposits, and **`mintFee`** now calculates fees since **`lastFeeCollected`** using the new, higher rate, incorrectly applying it to the period before the rate change.

**Impact:** 

The impact is twofold:

- An increased **`feePerSecond`** results in excessively high fees charged for the period before the update.
- A decreased **`feePerSecond`** leads to lower-than-expected fees for the same duration.

**Tools Used:** 

Manual Analysis

**Recommendation:** 

Ensure the fees are accurately accounted for at their respective rates by updating **`lastFeeCollected`** to the current timestamp prior to altering the **`feePerSecond`**. This can be achieved by invoking **`mintFee`** within the **`updateFeePerSecond`** function to settle all pending fees first:

```solidity
function updateFeePerSecond(uint256 feePerSecond) external onlyOwner {
	self.vault.mintFee();
        _store.feePerSecond = feePerSecond;
        emit FeePerSecondUpdated(feePerSecond);
    }
```

</details>

---
