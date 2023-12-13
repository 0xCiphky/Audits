# Temp

---

## About

---

## Scope

---

## Findings Summary

| ID  | Title                            | Severity   |
|-----|----------------------------------|------------|
| [H01](#h01---xxx) | When withdrawalBatchDuration is set to zero lenders can withdraw more then allocated to a batch                              | High       |
| [H02](#h02---xxx) | Create escrow parameters are mixed up, giving the sanctioned user the borrower role                              | High       |
| [H03](#h03---xxx) | Borrower closing market while secondsRemainingWithPenalty isn’t zero will lead to lenders not able to fully withdraw                              | High       |
| [H04](#h04---xxx) | Inability to Adjust Market Capacity and Close Market                              | High       |
| [H05](#h05---xxx) | Lenders can avoid being blocked and keep earning fees                              | High       |
| [M01](#m01---xxx) | create2 and create return value not checked                              | Medium     |
| [M02](#m02---xxx) | max/min constraints are not enforced on annualInterestBips after deployment                              | Medium     |
| [M03](#m03---xxx) | Sanctioned lender will still accrue interest contrary to the docs                              | Medium     |

---

## Detailed Findings

<details>
  <summary><a id="h01---xxx"></a>[H01] - When withdrawalBatchDuration is set to zero lenders can withdraw more then allocated to a batch</summary>
  
  <br>

**Severity:** High

**Summary:** 

The Wildcat protocol utilizes a withdrawal cycle where lenders call queueWithdrawals which then goes through a set amount of time (withdrawal duration period) before a withdrawal can be executed (if the protocol has enough funds to cover the withdrawal). Withdrawal requests that could not be fully honored at the end of their withdrawal cycle are batched together, marked as expired withdrawals, and added to the withdrawal queue. These batches are tracked using the time of expiry, and when assets are returned to a market with a non-zero withdrawal queue, assets are immediately routed to the unclaimed withdrawals pool and can subsequently be claimed by lenders with the oldest expired withdrawals first.

**Vulnerability Details:** 

The withdrawalBatchDuration can be set to zero so lenders do not have to wait before being able to withdraw funds from the market; however, this can cause issues where lenders in a batch can withdraw more than their pro-rata share of the batch's paid assets.

A lender calls queueWithdrawal first to initiate the withdrawal; this will place it in a batch respective to its expiry.

```solidity
function queueWithdrawal(uint256 amount) external nonReentrant {
        MarketState memory state = _getUpdatedState();

        ...

        // If there is no pending withdrawal batch, create a new one.
        if (state.pendingWithdrawalExpiry == 0) {
            state.pendingWithdrawalExpiry = uint32(block.timestamp + withdrawalBatchDuration);
            emit WithdrawalBatchCreated(state.pendingWithdrawalExpiry);
        }
        // Cache batch expiry on the stack for gas savings.
        uint32 expiry = state.pendingWithdrawalExpiry;

        WithdrawalBatch memory batch = _withdrawalData.batches[expiry];

        // Add scaled withdrawal amount to account withdrawal status, withdrawal batch and market state.
        _withdrawalData.accountStatuses[expiry][msg.sender].scaledAmount += scaledAmount;
        batch.scaledTotalAmount += scaledAmount;
        state.scaledPendingWithdrawals += scaledAmount;

        emit WithdrawalQueued(expiry, msg.sender, scaledAmount);

        // Burn as much of the withdrawal batch as possible with available liquidity.
        uint256 availableLiquidity = batch.availableLiquidityForPendingBatch(state, totalAssets());
        if (availableLiquidity > 0) {
            _applyWithdrawalBatchPayment(batch, state, expiry, availableLiquidity);
        }

        // Update stored batch data
        _withdrawalData.batches[expiry] = batch;

        // Update stored state
        _writeState(state);
    }
```

Now once the withdrawalBatchDuration has passed, a lender can call executeWithdrawal to finalize the withdrawal. This will grab the batch and let the lender withdraw a percentage of the batch if the batch is not fully paid or all funds if it is fully paid.

```solidity
function executeWithdrawal(address accountAddress, uint32 expiry) external nonReentrant returns (uint256) {
        if (expiry > block.timestamp) {
            revert WithdrawalBatchNotExpired();
        }
        MarketState memory state = _getUpdatedState();

        WithdrawalBatch memory batch = _withdrawalData.batches[expiry];
        AccountWithdrawalStatus storage status = _withdrawalData.accountStatuses[expiry][accountAddress];

        uint128 newTotalWithdrawn =
            uint128(MathUtils.mulDiv(batch.normalizedAmountPaid, status.scaledAmount, batch.scaledTotalAmount));
        uint128 normalizedAmountWithdrawn = newTotalWithdrawn - status.normalizedAmountWithdrawn;
        status.normalizedAmountWithdrawn = newTotalWithdrawn;
        state.normalizedUnclaimedWithdrawals -= normalizedAmountWithdrawn;

        ...

        // Update stored state
        _writeState(state);

        return normalizedAmountWithdrawn;
    }
```

Let's look at how this percentage is determined: the newTotalWithdrawn function determines a lender's available withdrawal amount by multiplying the normalizedAmountPaid with the scaledAmount and dividing the result by the batch's scaledTotalAmount. This ensures that each lender in the batch can withdraw an even amount of the available funds in the batch depending on their scaledAmount.

```solidity
 uint128 newTotalWithdrawn =
            uint128(MathUtils.mulDiv(batch.normalizedAmountPaid, status.scaledAmount, batch.scaledTotalAmount));
```

This works fine when withdrawalBatchDuration is set over zero, as the batch values (except normalizedAmountPaid) are finalized. However, when set to zero, we can end up with lenders in a batch being able to withdraw more than normalizedAmountPaid in that batch, potentially violating protocol invariants.

Consider the following scenario:

There is only 5 tokens available to burn

Lender A calls queueWithdrawal with 5 and executeWithdrawal instantly.

```solidity
newTotalWithdrawn = (normalizedAmountPaid) * (scaledAmount) / scaledTotalAmount

newTotalWithdrawn = 5 * 5 = 25 / 5 = 5
```

Lender A was able to fully withdraw.

Lender B comes along and calls queueWithdrawal with 5 and executeWithdrawal instantly in the same block.

This will add to the same batch as lender A as it is the same expiry.

Now let's look at newTotalWithdrawn for Lender B.

```solidity
newTotalWithdrawn = (normalizedAmountPaid) * (scaledAmount) / scaledTotalAmount

newTotalWithdrawn = 5 * 5 = 25 / 10 = 2.5
```

Lets see what the batch looks like now

- Lender A was able to withdraw 5 tokens in the batch

- Lender B was able to withdraw 2.5 tokens in the batch

- The batch.normalizedAmountPaid is 5, meaning the Lenders' withdrawal amount surpassed the batch's current limit.

**Impact:** 

This will break the following invariant in the protocol:

“Withdrawal execution can only transfer assets that have been counted as paid assets in the corresponding batch, i.e. lenders with withdrawal requests can not withdraw more than their pro-rata share of the batch's paid assets.”

It will also mean that funds reserved for other batches may not be able to be fulfilled even if the batch's normalizedAmountPaid number shows that it should be able to.

**Tools Used:** 

- Manual analysis
- Foundry

**Recommendation:** 

Review the protocol's withdrawal mechanism and consider adjusting the behaviour of withdrawals when withdrawalBatchDuration is set to zero to ensure that lenders cannot withdraw more than their pro-rata share of the batch's paid assets.

</details>

<details>
  <summary><a id="h02---xxx"></a>[H02] - Create escrow parameters are mixed up, giving the sanctioned user the borrower role</summary>
  
  <br>

**Severity:** High

**Summary:** 

  The Wildcat Protocol implements the ability to deploy an escrow contract between the borrower of a market and the lender in question in the event that a lender address is sanctioned. This is done by the borrower calling the nukeFromOrbit function with the borrower's address. If the lender is indeed sanctioned, it creates an escrow contract, transfers the vault balance corresponding to the lender from the market to the escrow, erases the lender's market token balance, and blocks them from any further interaction with the market itself.

However, an issue arises from the mixed-up parameters in the createEscrow function, which switches the roles of the borrower and the lender within the created escrow.

**Vulnerability Details:** 

The createEscrow function is used in two places in the protocol, in the executeWithdrawal and the _blockAccount functions. Both functions implement it in the following way:

```solidity
// _blockAccount
address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(accountAddress, borrower, address(this));

// executeWithdrawal
address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(accountAddress, borrower, address(asset));
```

Now let's look at the createEscrow function and how it's implemented. The issue is the way we order the parameters. In the createEscrow function, we can see that the order is borrower, account, asset, whereas in the _blockAccount and executeWithdrawal functions, it is accountAddress, borrower, address(asset).

```solidity
function createEscrow(
    address borrower,
    address account,
    address asset
  ) public override returns (address escrowContract) {
    if (!IWildcatArchController(archController).isRegisteredMarket(msg.sender)) {
      revert NotRegisteredMarket();
    }
```
As we can see, the borrower and sanctioned lenders are in the incorrect order, meaning for the created escrow, they will switch roles. This would allow the sanctioned user to override the sanction and release the sanctioned funds.

The lender can call overrideSanction in WildcatSanctionsSentinel, this should not normally work however the roles are switched and the lenders address is the borrower in the escrow and vice versa.

```solidity
function overrideSanction(address account) public override {
    sanctionOverrides[msg.sender][account] = true;
    emit SanctionOverride(msg.sender, account);
  }
```

Now the lender can call releaseEscrow which will pass.

**Proof of concept:** 

```solidity
function test_nukeFromOrbit_WrongEscrowAddress() external {
        _deposit(alice, 1e18);
        // wrong way to get escrow address
        address escrowWrong = sanctionsSentinel.getEscrowAddress(alice, borrower, address(market));
        // correct way to get escrow address
        address escrow10 = sanctionsSentinel.getEscrowAddress(borrower, alice, address(market));
        // sanction alice
        sanctionsSentinel.sanction(alice);
        // nuke alice
        market.nukeFromOrbit(alice);
        // check alice role
        assertEq(uint256(market.getAccountRole(alice)), uint256(AuthRole.Blocked), "account role should be Blocked");
        // check sanction override mapping
        assertEq(sanctionsSentinel.sanctionOverrides(borrower, escrowWrong), false);
        assertEq(sanctionsSentinel.sanctionOverrides(alice, escrowWrong), true);
    }
```

**Impact:** 

The borrower and lender roles will be switched in the created escrow. A sanctioned lender can release their sanctioned funds without the borrower authorization or the sanction being overturned.

**Tools Used:** 

- Manual analysis
- Foundry

**Recommendation:** 

Use the correct order for the parameters in createEscrow in the _blockAccount and executeWithdrawal functions.

```solidity
// _blockAccount
address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(borrower, accountAddress, address(this));

// executeWithdrawal
address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(borrower, accountAddress, address(asset));
```

</details>

<details>
  <summary><a id="h03---xxx"></a>[H03] - Borrower closing market while secondsRemainingWithPenalty isn’t zero will lead to lenders not able to fully withdraw</summary>
  
  <br>

**Severity:** High

**Summary:** 

The protocol's health is monitored through a reserve ratio, representing the percentage of the market's supply required to remain within the market for redemption. Falling below this threshold results in market delinquency.

When a market becomes delinquent, a penalty rate is applied to the base rate as long as the grace tracker exceeds the grace period. The grace period is dynamic, counting down to zero when delinquency is resolved, and only then does the penalty APR cease.

```solidity
function updateTimeDelinquentAndGetPenaltyTime(
    MarketState memory state,
    uint256 delinquencyGracePeriod,
    uint256 timeDelta
  ) internal pure returns (uint256 /* timeWithPenalty */) {
    // Seconds in delinquency at last update
    uint256 previousTimeDelinquent = state.timeDelinquent;

    if (state.isDelinquent) {
      // Since the borrower is still delinquent, increase the total
      // time in delinquency by the time elapsed.
      state.timeDelinquent = (previousTimeDelinquent + timeDelta).toUint32();

      // Calculate the number of seconds the borrower had remaining
      // in the grace period.
      uint256 secondsRemainingWithoutPenalty = delinquencyGracePeriod.satSub(
        previousTimeDelinquent
      );

      // Penalties apply for the number of seconds the market spent in
      // delinquency outside of the grace period since the last update.
      return timeDelta.satSub(secondsRemainingWithoutPenalty);
    }

    // Reduce the total time in delinquency by the time elapsed, stopping
    // when it reaches zero.
    state.timeDelinquent = previousTimeDelinquent.satSub(timeDelta).toUint32();

    // Calculate the number of seconds the old timeDelinquent had remaining
    // outside the grace period, or zero if it was already in the grace period.
    uint256 secondsRemainingWithPenalty = previousTimeDelinquent.satSub(delinquencyGracePeriod);

    // Only apply penalties for the remaining time outside of the grace period.
    return MathUtils.min(secondsRemainingWithPenalty, timeDelta);
  }
```

A borrower can close a market in the event that they have finished utilizing the funds. When a vault is closed, sufficient assets must be repaid to increase the reserve ratio to 100%, after which interest ceases to accrue, and no further parameter adjustment or borrowing is possible.

However, an issue arises when a borrower closes a market while the secondsRemainingWithPenalty is still active. This results in the delinquency fee persisting, leading to an increase in the scale factor, which should remain constant after market closure, as the borrower has repaid all funds at that rate.

```solidity
function closeMarket() external onlyController nonReentrant {
        MarketState memory state = _getUpdatedState();
        state.annualInterestBips = 0;
        state.isClosed = true;
        state.reserveRatioBips = 0;
        if (_withdrawalData.unpaidBatches.length() > 0) {
            revert CloseMarketWithUnpaidWithdrawals();
        }
        uint256 currentlyHeld = totalAssets();
        uint256 totalDebts = state.totalDebts();
        if (currentlyHeld < totalDebts) {
            // Transfer remaining debts from borrower
            asset.safeTransferFrom(borrower, address(this), totalDebts - currentlyHeld);
        } else if (currentlyHeld > totalDebts) {
            // Transfer excess assets to borrower
            asset.safeTransfer(borrower, currentlyHeld - totalDebts);
        }
        _writeState(state);
        emit MarketClosed(block.timestamp);
    }
```

Consequently, the increased scale factor means that the total funds in the market won't cover all lenders, and lenders exiting closer to the end may not be able to fully withdraw their funds.

**Proof Of Concept:** 

```solidity
function test_closeMarket_WhileStillInPenalty() external asAccount(address(controller)) {
        asset.mint(address(borrower), type(uint128).max);
        assertEq(market.currentState().isDelinquent, false);
        // alice deposit
        _deposit(alice, 1e18);
        // borrow 80% of deposits
        _borrow(8e17);
        // request withdrawal to put borrower in penalty
        _requestWithdrawal(alice, 1e18);
        // borrower now delinquent
        assertEq(market.currentState().isDelinquent, true);
        // fast forward grace period plus 5 days
        fastForward(parameters.delinquencyGracePeriod + 5 days);
        // borrower transfer  deposits
        startPrank(borrower);
        asset.transfer(address(market), 1e18);
        stopPrank();
        market.updateState();
        // borrower close market
        startPrank(borrower);
        asset.approve(address(market), 20e17);
        stopPrank();
        market.closeMarket();
        // check final scale factor
        uint112 FinalScaleFactor = market.currentState().scaleFactor;
        assertEq(market.currentState().isClosed, true);
        fastForward(10 days);
        // check scale factor 10 days after close market
        assertGt(market.currentState().scaleFactor, FinalScaleFactor);
    }
```

**Impact:** 

The Scale factor will continue to increase after the market was closed by the borrower, meaning lenders who withdraw closer to the end will not be able to fully withdraw from the market, resulting in a loss of funds.

**Tools Used:** 

- Manual analysis
- Foundry

**Recommendation:** 

Reset the grace tracker to zero upon market closure to prevent the delinquency fee from persisting and causing an increase in the scale factor.

```solidity
function closeMarket() external onlyController nonReentrant {
        MarketState memory state = _getUpdatedState();
        state.annualInterestBips = 0;
        state.isClosed = true;
        state.reserveRatioBips = 0;
        state.timeDelinquent = 0; // add here
        if (_withdrawalData.unpaidBatches.length() > 0) {
            revert CloseMarketWithUnpaidWithdrawals();
        }
        uint256 currentlyHeld = totalAssets();
        uint256 totalDebts = state.totalDebts();
        if (currentlyHeld < totalDebts) {
            // Transfer remaining debts from borrower
            asset.safeTransferFrom(borrower, address(this), totalDebts - currentlyHeld);
        } else if (currentlyHeld > totalDebts) {
            // Transfer excess assets to borrower
            asset.safeTransfer(borrower, currentlyHeld - totalDebts);
        }
        _writeState(state);
        emit MarketClosed(block.timestamp);
    }
```

</details>

<details>
  <summary><a id="h04---xxx"></a>[H04] - Inability to Adjust Market Capacity and Close Market</summary>
  
  <br>

**Severity:** High

**Summary:** 

Borrowers have the capability to modify a market's maximum capacity and interest APR in the Wildcat Protocol. The code implements the setMaxTotalSupply and setAnnualInterestBips functions, both equipped with an onlyController modifier to restrict access to the controller contract.

This is fine for setAnnualInterestBips as its invoked in the WildcatMarketController contract however the setMaxTotalSupply function is not meaning the maximum supply cannot be adjusted. The same issue occurs with the closeMarket function in the WildcatMarket contract meaning the borrower will not be able to close the market.

**Vulnerability Details:** 

The setMaxTotalSupply function enforces access control to permit only the controller contract to invoke it. However, the controller contract does not call this function, rendering it unusable and preventing adjustments to the maximum supply capacity.

```solidity
function setMaxTotalSupply(uint256 _maxTotalSupply) external onlyController nonReentrant {
        MarketState memory state = _getUpdatedState();

        if (_maxTotalSupply < state.totalSupply()) {
            revert NewMaxSupplyTooLow();
        }

        state.maxTotalSupply = _maxTotalSupply.toUint128();
        _writeState(state);
        emit MaxTotalSupplyUpdated(_maxTotalSupply);
    }
```

A similar issue arises with the closeMarket function, which also employs the onlyController modifier. Consequently, borrowers are currently unable to close markets.

```solidity
function closeMarket() external onlyController nonReentrant {
        MarketState memory state = _getUpdatedState();
        state.annualInterestBips = 0;
        state.isClosed = true;
        state.reserveRatioBips = 0;
        if (_withdrawalData.unpaidBatches.length() > 0) {
            revert CloseMarketWithUnpaidWithdrawals();
        }
        uint256 currentlyHeld = totalAssets();
        uint256 totalDebts = state.totalDebts();
        if (currentlyHeld < totalDebts) {
            // Transfer remaining debts from borrower
            asset.safeTransferFrom(borrower, address(this), totalDebts - currentlyHeld);
        } else if (currentlyHeld > totalDebts) {
            // Transfer excess assets to borrower
            asset.safeTransfer(borrower, currentlyHeld - totalDebts);
        }
        _writeState(state);
        emit MarketClosed(block.timestamp);
    }
```
**Proof of concept:**

```solidity
function closeMarket() external onlyController nonReentrant {
        MarketState memory state = _getUpdatedState();
        state.annualInterestBips = 0;
        state.isClosed = true;
        state.reserveRatioBips = 0;
        if (_withdrawalData.unpaidBatches.length() > 0) {
            revert CloseMarketWithUnpaidWithdrawals();
        }
        uint256 currentlyHeld = totalAssets();
        uint256 totalDebts = state.totalDebts();
        if (currentlyHeld < totalDebts) {
            // Transfer remaining debts from borrower
            asset.safeTransferFrom(borrower, address(this), totalDebts - currentlyHeld);
        } else if (currentlyHeld > totalDebts) {
            // Transfer excess assets to borrower
            asset.safeTransfer(borrower, currentlyHeld - totalDebts);
        }
        _writeState(state);
        emit MarketClosed(block.timestamp);
    }
```

```solidity
function test_ChangeMaxCapacity() external {
        // try to change max capacity
        vm.expectRevert(IMarketEventsAndErrors.NotController.selector);
        market.setMaxTotalSupply(100e18);
    }
```

**Impact:** 

A borrower will not be able to Adjust Market Capacity or Close the Market.

**Tools Used:** 

- Manual analysis
- Foundry

**Recommendation:** 

Add functions in the WildcatMarketController contract that invoke the setMaxTotalSupply and closeMarket function so a borrower is able to Adjust Market Capacity and Close the Market.

</details>

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

---
