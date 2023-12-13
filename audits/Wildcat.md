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
  <summary><a id="h03---xxx"></a>[H03] - XXX</summary>
  
  <br>

  **Severity:** High

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

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
