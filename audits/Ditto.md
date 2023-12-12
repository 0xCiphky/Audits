# Ditto - DittoETH

---

## About

The system mints pegged assets (stablecoins) using an orderbook, using over-collateralized staked ETH.

---

## Findings Summary

| ID  | Title                            | Severity   |
|-----|----------------------------------|------------|
| [H01](#h01---xxx) | Users can avoid liquidation while being under the primary liquidation ratio if on the last short record                              | High       |
| [H02](#h02---xxx) | Flagger Ids are reused too early, potentially blocking flaggers from liquidating in there allocated time                              | High       |
| [M01](#m01---xxx) | Combining shorts can incorrectly reset the shorts flag                              | Medium     |
| [L01](#l01---xxx) | Last short does not reset liquidation flag after user gets fully liquidated, meaning healthy position will still be flagged if another order gets filled                              | Low/Info   |
| [L02](#l02---xxx) | Last short does not reset liquidation flag after user exits position fully, meaning healthy position will still be flagged if another order gets filled                              | Low/Info   |
| [L03](#l03---xxx) | Partial filled short does not reset liquidation flag after user gets fully liquidated, meaning healthy position will still be flagged if the rest of the order gets filled                              | Low/Info   |
| [L04](#l04---xxx) | Partial filled short does not reset flag after user exits position fully, meaning healthy position will still be flagged if the rest of the order gets filled                              | Low/Info   |
| [L05](#l05---xxx) | Protocol doesn’t take into account RETH/STETH requirements                              | Low/Info   |
| [L06](#l06---xxx) | User can accidentally or maliciously combine the same short                              | Low/Info   |
| [L07](#l07---xxx) | Users who are flagged and get back to a healthy ratio through price increase are still flagged contrary to the docs                              | Low/Info   |
| [L08](#l08---xxx) | collateral ratio can never be the max                              | Low/Info   |
| [L09](#l09---xxx) | Event in secondaryLiquidation could be misused to show false liquidations                              | Low/Info   |

---

## Detailed Findings

<details>
  <summary><a id="h01---xxx"></a>[H01] - Users can avoid liquidation while being under the primary liquidation ratio if on the last short record</summary>
  
  <br>

  **Severity:** High

  **Summary:** 

  The protocol permits users to maintain up to 254 concurrent short records. When this limit is reached, any additional orders are appended to the final position, rather than creating a new one. A short record is subject to flagging if it breaches the primary liquidation ratio set by the protocol, leading to potential liquidation if it remains below the threshold for a predefined period.

The vulnerability emerges from the dependency of liquidation times on the **`updatedAt`** value of shorts. For the last short record, the appending of any new orders provides an alternative pathway for updating the **`updatedAt`** value of shorts, enabling users to circumvent liquidation by submitting minimal shorts to block liquidation by adjusting the time difference, thus avoiding liquidation even when they do not meet the collateral requirements for a healthy state.

  **Vulnerability Details:** 

lets take a look at the code to see how this works.
1. **Flagging of Short Record:**
    - The **`flagShort`** function allows a short to be flagged if it's under **`primaryLiquidationCR`**, subsequently invoking **`setFlagger`** which updates the short's **`updatedAt`** timestamp to the current time.

```solidity
function flagShort(address asset, address shorter, uint8 id, uint16 flaggerHint)
        external
        isNotFrozen(asset)
        nonReentrant
        onlyValidShortRecord(asset, shorter, id)
    {
        // initial code

        short.setFlagger(cusd, flaggerHint);
        emit Events.FlagShort(asset, shorter, id, msg.sender, adjustedTimestamp);
    }
```

1. **Liquidation Eligibility Check:**
    - The **`_canLiquidate`** function assesses whether the flagged short is still under **`primaryLiquidationCR`** after a certain period and if it's eligible for liquidation, depending on the **`updatedAt`** timestamp and various liquidation time frames.

```solidity
function _canLiquidate(MTypes.MarginCallPrimary memory m)
        private
        view
        returns (bool)
    {
       // Initial code

        uint256 timeDiff = LibOrders.getOffsetTimeHours() - m.short.updatedAt;
        uint256 resetLiquidationTime = LibAsset.resetLiquidationTime(m.asset);

        if (timeDiff >= resetLiquidationTime) {
            return false;
        } else {
            uint256 secondLiquidationTime = LibAsset.secondLiquidationTime(m.asset);
            bool isBetweenFirstAndSecondLiquidationTime = timeDiff
                > LibAsset.firstLiquidationTime(m.asset) && timeDiff <= secondLiquidationTime
                && s.flagMapping[m.short.flaggerId] == msg.sender;
            bool isBetweenSecondAndResetLiquidationTime =
                timeDiff > secondLiquidationTime && timeDiff <= resetLiquidationTime;
            if (
                !(
                    (isBetweenFirstAndSecondLiquidationTime)
                        || (isBetweenSecondAndResetLiquidationTime)
                )
            ) {
                revert Errors.MarginCallIneligibleWindow();
            }

            return true;
        }
    }
}
```

1. **Short Record Merging:**
    - For the last short record, the **`fillShortRecord`** function combines new matched shorts with the existing one, invoking the **`merge`** function, which updates the **`updatedAt`** value to the current time.

```solidity
function fillShortRecord(
        address asset,
        address shorter,
        uint8 shortId,
        SR status,
        uint88 collateral,
        uint88 ercAmount,
        uint256 ercDebtRate,
        uint256 zethYieldRate
    ) internal {
        AppStorage storage s = appStorage();

        uint256 ercDebtSocialized = ercAmount.mul(ercDebtRate);
        uint256 yield = collateral.mul(zethYieldRate);

        STypes.ShortRecord storage short = s.shortRecords[asset][shorter][shortId];
        if (short.status == SR.Cancelled) {
            short.ercDebt = short.collateral = 0;
        }

        short.status = status;
        LibShortRecord.merge(
            short,
            ercAmount,
            ercDebtSocialized,
            collateral,
            yield,
            LibOrders.getOffsetTimeHours()
        );
    }
```

- In the merge function we see that we update the updatedAt value to creationTime which is  LibOrders.getOffsetTimeHours().

```solidity
function merge(
        STypes.ShortRecord storage short,
        uint88 ercDebt,
        uint256 ercDebtSocialized,
        uint88 collateral,
        uint256 yield,
        uint24 creationTime
    ) internal {
        // Resolve ercDebt
        ercDebtSocialized += short.ercDebt.mul(short.ercDebtRate);
        short.ercDebt += ercDebt;
        short.ercDebtRate = ercDebtSocialized.divU64(short.ercDebt);
        // Resolve zethCollateral
        yield += short.collateral.mul(short.zethYieldRate);
        short.collateral += collateral;
        short.zethYieldRate = yield.divU80(short.collateral);
        // Assign updatedAt
        short.updatedAt = creationTime;
    }
```

- This means that even if the position was flagged and is still under the **`primaryLiquidationCR`**, it cannot be liquidated as the **`updatedAt`** timestamp has been updated, making the time difference not big enough.

<details>
  <summary><b>Click to expand Proof of Concept</b></summary>

  ```solidity
    function testShortAvoidLiquidation() public {
        // fill  shorts (up to 254)
        for (uint i; i < 253; i++) {
            fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT * 5, sender);
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT * 5, receiver);
        } 
        
        // check users last shortrecord
        assertTrue(getShortRecord(sender, 254).status == SR.FullyFilled);

        // price drop
        skipTimeAndSetEth(1 hours, 2000 ether);

        // flag short
        vm.prank(receiver);
        diamond.flagShort(asset, sender, 254, Constants.HEAD);

        // check flag
        assertTrue(getShortRecord(sender, 254).flaggerId == 1);

        // skip time to primary liquidation time
        skipTimeAndSetEth(11 hours, 2000 ether);

        // User matches new min short (added to last spot)
        fundLimitShortOpt(DEFAULT_PRICE * 2, DEFAULT_AMOUNT  , sender);
        fundLimitBidOpt(DEFAULT_PRICE * 2, DEFAULT_AMOUNT  , receiver);

        // flagger tries to liquidate short in eligible window
        fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT * 6, extra);
        vm.startPrank(receiver);
        vm.expectRevert(Errors.MarginCallIneligibleWindow.selector);
        diamond.liquidate(
            asset, sender, 254, shortHintArrayStorage
        );
        vm.stopPrank();
    }
```
</details>

  **Impact:** 

  This allows a user with a position under the primaryLiquidationCR to avoid primary liquidation even if the short is in the valid time ranges for liquidation.

  **Tools Used:** 
  - Manual analysis
  - Foundry

  **Recommendation:** 

  Impose stricter conditions for updating the last short record when the position is flagged and remains under the **`primaryLiquidationCR`** post-merge, similar to how the **`combineShorts`** function works.

```solidity
function createShortRecord(
        address asset,
        address shorter,
        SR status,
        uint88 collateral,
        uint88 ercAmount,
        uint64 ercDebtRate,
        uint80 zethYieldRate,
        uint40 tokenId
    ) internal returns (uint8 id) {
        AppStorage storage s = appStorage();

        // Initial code

        } else {
            // All shortRecordIds used, combine into max shortRecordId
            id = Constants.SHORT_MAX_ID;
            fillShortRecord(
                asset,
                shorter,
                id,
                status,
                collateral,
                ercAmount,
                ercDebtRate,
                zethYieldRate
            );

				// If the short was flagged, ensure resulting c-ratio > primaryLiquidationCR
		        if (Constants.SHORT_MAX_ID.shortFlagExists) {
	                if (
	                    Constants.SHORT_MAX_ID.getCollateralRatioSpotPrice(
	                        LibOracle.getSavedOrSpotOraclePrice(_asset)
	                    ) < LibAsset.primaryLiquidationCR(_asset)
	                ) revert Errors.InsufficientCollateral();
	                // Resulting combined short has sufficient c-ratio to remove flag
	                Constants.SHORT_MAX_ID.resetFlag();
		        }
            }
    }
```

</details>

<details>
  <summary><a id="h02---xxx"></a>[H02] - Flagger Ids are reused too early, potentially blocking flaggers from liquidating in there allocated time</summary>
  
  <br>

  **Severity:** High

  **Summary:** 

  The protocol enables users to flag positions that fall below the primary collateral ratio. Subsequently, the shorter is granted a time frame to restore their position above this ratio to avoid liquidation. If the position remains below the primary collateral ratio, the flagger attains the exclusive right to liquidate it before anyone else.

  **Vulnerability Details:** 

  To optimize the process, the protocol reuses flagger IDs. However, a flaw exists in the protocol where a flagger ID is available for reuse after the firstLiquidationTime instead of after the secondLiquidationTime.

```solidity
//@dev re-use an inactive flaggerId
if (timeDiff > LibAsset.firstLiquidationTime(cusd)) {
   delete s.assetUser[cusd][flaggerToReplace].g_flaggerId;
   short.flaggerId = flagStorage.g_flaggerId = flaggerHint;
```

This premature reuse of the flagger ID can block a flagger from liquidating a position during their allocated slot, which spans between firstLiquidationTime and secondLiquidationTime.

```solidity
uint256 secondLiquidationTime = LibAsset.secondLiquidationTime(m.asset);
            bool isBetweenFirstAndSecondLiquidationTime = timeDiff
                > LibAsset.firstLiquidationTime(m.asset) && timeDiff <= secondLiquidationTime
                && s.flagMapping[m.short.flaggerId] == msg.sender;
```

<details>
  <summary><b>Click to expand Proof of Concept</b></summary>

  ```solidity
  function testShortFlagReusedTooEarly() public {
        skipTimeAndSetEth(2 hours, 4000 ether);

        // Create short 1
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);
        // Create short 2
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);

        // match short 1
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        // match short 2
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        // extra Ask for liquidation
        fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT , extra);

        // skip time, price fall
        skipTimeAndSetEth(2 hours, 2000 ether);

        // Extra user flag short 1
        vm.prank(extra);
        diamond.flagShort(asset, sender, Constants.SHORT_STARTING_ID, Constants.HEAD);

        // skip user grace period
        skipTimeAndSetEth(11 hours, 2000 ether);
        
        // receiver flags short 2
        vm.prank(receiver);
        diamond.flagShort(asset, sender, Constants.SHORT_STARTING_ID + 1, Constants.HEAD);

         vm.startPrank(extra);
        // extra user tries to liquidate short 1 in the valid time range but flag is reused so fails
        vm.expectRevert(Errors.MarginCallIneligibleWindow.selector);
        diamond.liquidate(
            asset, sender, Constants.SHORT_STARTING_ID, shortHintArrayStorage
        );
        vm.stopPrank();
    }
```
</details>

  **Impact:** 

  Flaggers is unable to liquidate short positions during their designated time slots

  **Tools Used:** 

  - Manual Analysis
  - Foundry

  **Recommendation:** 

  Ensure that flagger IDs are reused only after the secondLiquidationTime.

```solidity
if (timeDiff > LibAsset.secondLiquidationTime(cusd)) {
   delete s.assetUser[cusd][flaggerToReplace].g_flaggerId;
   short.flaggerId = flagStorage.g_flaggerId = flaggerHint;
```

</details>

---

<details>
  <summary><a id="m01---xxx"></a>[M01] - Combining shorts can incorrectly reset the shorts flag</summary>
  
  <br>

  **Severity:** Medium

  **Summary:** 

  The protocol allows users to combine multiple short positions into one as long as the combined short stays above the primary collateral ratio. The function is also able to reset an active flag from any of the combined shorts if the final ratio is above the primaryLiquidationCR.

The issue is that the combineShorts function does not call updateErcDebt, which is called in every other function that is able to reset a shorts flag. This means that if the debt is outdated the final combined short could incorrectly reset the flag putting the position on a healthy ratio when it really isn’t. This would also mean that it will have to be reflagged and go through the timer again before it can be liquidated.

  **Vulnerability Details:** 

  The combine shorts function merges all short records into the short at position id[0]. Focusing on the debt aspect it adds up the total debt and calculates the ercDebtSocialized of all positions except for the first.

```solidity
      {
      uint88 currentShortCollateral = currentShort.collateral;
      uint88 currentShortErcDebt = currentShort.ercDebt;
      collateral += currentShortCollateral;
      ercDebt += currentShortErcDebt;
      yield += currentShortCollateral.mul(currentShort.zethYieldRate);
      ercDebtSocialized += currentShortErcDebt.mul(currentShort.ercDebtRate);
      }
```

It then merges this total to the first position using the merge function and this will give us the combined short.

```solidity
// Merge all short records into the short at position id[0]
        firstShort.merge(ercDebt, ercDebtSocialized, collateral, yield, c.shortUpdatedAt);
```

Finally we check if the position had an active flag and if it did, we check if the new combined short is in a healthy enough state to reset the flag, if not the whole function reverts.

```solidity
        // If at least one short was flagged, ensure resulting c-ratio > primaryLiquidationCR
        if (c.shortFlagExists) {
            if (
                firstShort.getCollateralRatioSpotPrice(
                    LibOracle.getSavedOrSpotOraclePrice(_asset)
                ) < LibAsset.primaryLiquidationCR(_asset)
            ) revert Errors.InsufficientCollateral();
            // Resulting combined short has sufficient c-ratio to remove flag
            firstShort.resetFlag();
        }
```

As you can see the updateErcDebt function is not called anywhere in the function meaning the flag could be reset with outdated values.

  **Impact:** 

  A short could have its flag incorrectly reset and reset the timer. This is not good for the protocol as it will have a unhealthy short for a longer time.

  **Tools Used:** 

  - Manual analysis
  - Foundry

  **Recommendation:** 

  Call updateErcDebt on the short once it is combined in the combineShorts function to ensure the collateral ratio is calculated with the most up to date values.

```solidity
    function combineShorts(address asset, uint8[] memory ids)
        external
        isNotFrozen(asset)
        nonReentrant
        onlyValidShortRecord(asset, msg.sender, ids[0])
    {
        // Initial code

        // Merge all short records into the short at position id[0]
        firstShort.merge(ercDebt, ercDebtSocialized, collateral, yield, c.shortUpdatedAt);

        firstShort.updateErcDebt(asset); // update debt here before checking flag

        // If at least one short was flagged, ensure resulting c-ratio > primaryLiquidationCR
        if (c.shortFlagExists) {
            if (
                firstShort.getCollateralRatioSpotPrice(
                    LibOracle.getSavedOrSpotOraclePrice(_asset)
                ) < LibAsset.primaryLiquidationCR(_asset)
            ) revert Errors.InsufficientCollateral();
            // Resulting combined short has sufficient c-ratio to remove flag
            firstShort.resetFlag();
        }
        emit Events.CombineShorts(asset, msg.sender, ids);
    }
```

</details>

---

<details>
  <summary><a id="l01---xxx"></a>[L01] - Last short does not reset liquidation flag after user gets fully liquidated, meaning healthy position will still be flagged if another order gets filled</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  - The protocol permits users to maintain up to 254 concurrent short records. When this limit is reached, any additional orders are appended to the final position, rather than creating a new one.
- A short record is flagged if it falls below the primary liquidation ratio set by the protocol, signalling to the user that their position is nearing an unhealthy state. The user can resolve this by modifying the position to improve its health or by paying off the short and exiting the position.
- If a user is unable to get their their position to a healthy state by a certain time they can be liquidated.
- A vulnerability exists where, under specific circumstances, a user’s healthy position is flagged and can be instantly liquidated without warning.

  **Vulnerability Details:**

  - Consider the following scenario
    1. User A creates a short order, that gets matched and fills in the last short (ID 254).
    2. User A’s position falls below the primary liquidation ratio and is flagged by User B.
    3. User A’s position is fully liquidated by User B, with the flag remaining active post liquidation.
    4. Another order gets filled at a healthy ratio at the same ID but remains flagged.

<details>
  <summary><b>Click to expand Proof of Concept</b></summary>

  ```solidity
    function testLastShortLiqShort() public {
        skipTimeAndSetEth(2 hours, 4000 ether);
        // fill up shorts (up to 253)
        for (uint256 i; i < 252; i++) {
            fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        }

        // Create short 254
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT , sender);

        // create bid for short
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);

        //get short
        STypes.ShortRecord memory shortBeforeFlag = getShortRecord(sender, 254);

        //check short flag
        assertEq(shortBeforeFlag.flaggerId, 0);

        // fall in price
        skipTimeAndSetEth(2 hours, 2000 ether);

        // flag short
        vm.prank(extra);
        diamond.flagShort(asset, sender, 254, Constants.HEAD);

        //get short
        STypes.ShortRecord memory shortAfterFlag = getShortRecord(sender, 254);
        //check short flag
        assertGt(shortAfterFlag.flaggerId, 0);

        skipTimeAndSetEth(11 hours, 2000 ether);
        fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, extra);

          // liquidate short
        vm.prank(extra);
        diamond.liquidate(asset, sender, 254, shortHintArrayStorage);

        //get short
        STypes.ShortRecord memory shortAfterExit = getShortRecord(sender, 254);
        //check short flag
        assertGt(shortAfterExit.flaggerId, 0);

        //price recover back to initial
        skipTimeAndSetEth(2 hours, 4000 ether);

        // Create short 254
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT , sender);

        // create bid for short
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        //get short
        STypes.ShortRecord memory shortAfterMatch = getShortRecord(sender, 254);

        //check short flag
        assertGt(shortAfterMatch.flaggerId, 0);

        //price fall
        skipTimeAndSetEth(11 hours, 2000 ether);
        fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, extra);

        // liquidate short
        vm.prank(extra);
        diamond.liquidate(asset, sender, 254, shortHintArrayStorage);
    }
```
</details>

**Impact:** 

  - A healthy short is incorrectly flagged.
- If the new short falls below the primary liquidation ratio:
  - It cannot be flagged by another user until updatedAt (when short was filled) plus the reset time is reached.
  - It can be liquidated after updatedAt (when short was filled) plus the firstLiquidationTime till resetLiquidationTime even if it was never flagged.
  - Keep in mind the shorts updatedAt will be updated when the short gets filled so this will push the liquidation times up by the time diff (fillShort - flagged).
- The protocol gives users a grace period to reestablish their positions when they fall below the primary liquidation ratio, however in the following situation a user can be liquidated without warning (being flagged).
- A user is also unable to use certain protocol functionality (e.g. transfer his short).

**Tools Used:**

  - Manual Analysis
  - Foundry

**Recommendation:**

  - The liquidation process must reset the flag in full liquidations to ensure that users don’t start off with healthy positions flagged when the another order gets matched to the last short.

```solidity
if (m.short.ercDebt == m.ercDebtMatched) {
            // Full liquidation
            LibShortRecord.disburseCollateral(
                m.asset,
                m.shorter,
                m.short.collateral,
                m.short.zethYieldRate,
                m.short.updatedAt
            );
            LibShortRecord.deleteShortRecord(m.asset, m.shorter, m.short.id);
            if (!m.loseCollateral) {
                m.short.collateral -= decreaseCol;
                s.vaultUser[m.vault][m.shorter].ethEscrowed += m.short.collateral;
                s.vaultUser[m.vault][address(this)].ethEscrowed -= m.short.collateral;

		// reset flag here
		short.resetFlag()
            }
```

</details>

<details>
  <summary><a id="l02---xxx"></a>[L02] - Last short does not reset liquidation flag after user exits position fully, meaning healthy position will still be flagged if another order gets filled</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  - The protocol permits users to maintain up to 254 concurrent short records. When this limit is reached, any additional orders are appended to the final position, rather than creating a new one.
- A short record is flagged if it falls below the primary liquidation ratio set by the protocol, signalling to the user that their position is nearing an unhealthy state. The user can resolve this by modifying the position to improve its health or by paying off the short and exiting the position.
- A vulnerability exists where, under specific circumstances, a user’s healthy position is flagged and can be instantly liquidated without warning.

  **Vulnerability Details:**

  - Consider the following scenario
    1. User A creates a short order, that gets matched and fills in the last short (ID 254).
    2. User A’s position falls below the primary liquidation ratio and is flagged.
    3. User A calls **`exitShortErcEscrowed`** to pay off the position.
        1. The full amount was paid off but maybeResetFlag is not called.
    4. Another short order gets filled at a healthy ratio, creating the same short record (ID 254).

<details>
  <summary><b>Click to expand Proof of Concept</b></summary>

  ```solidity
    function testLastShortFExitPShort() public {
        skipTimeAndSetEth(2 hours, 4000 ether);
        // fill up shorts (up to 253)
        for (uint256 i; i < 252; i++) {
            fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        }

        // Create short 254
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT , sender);

        // create bid for short
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);

        //get short
        STypes.ShortRecord memory shortBeforeFlag = getShortRecord(sender, 254);

        //check short flag
        assertEq(shortBeforeFlag.flaggerId, 0);

        // fall in price
        skipTimeAndSetEth(2 hours, 2000 ether);

        // flag short
        vm.prank(extra);
        diamond.flagShort(asset, sender, 254, Constants.HEAD);

        //get short
        STypes.ShortRecord memory shortAfterFlag = getShortRecord(sender, 254);
        //check short flag
        assertGt(shortAfterFlag.flaggerId, 0);

        // exit short
        exitShortErcEscrowed(254, DEFAULT_AMOUNT, sender);

        //get short
        STypes.ShortRecord memory shortAfterExit = getShortRecord(sender, 254);
        //check short flag
        assertGt(shortAfterExit.flaggerId, 0);

        //price recover back to initial
        skipTimeAndSetEth(2 hours, 4000 ether);

        // Create short 254
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT , sender);

        // create bid for short
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        //get short
        STypes.ShortRecord memory shortAfterMatch = getShortRecord(sender, 254);

        //check short flag
        assertGt(shortAfterMatch.flaggerId, 0);

        //price fall
        skipTimeAndSetEth(11 hours, 2000 ether);
        fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, extra);

        // liquidate short
        vm.prank(extra);
        diamond.liquidate(asset, sender, 254, shortHintArrayStorage);
    }
```
</details>

**Impact:** 

- A healthy short is incorrectly flagged.
- If the new short falls below the primary liquidation ratio:
  - It cannot be flagged by another user until updatedAt (when short was filled) plus the reset time is reached.
  - It can be liquidated after updatedAt (when short was filled) plus the firstLiquidationTime till resetLiquidationTime even if it was never flagged.
  - Keep in mind the shorts updatedAt will be updated when the short gets filled so this will push the liquidation times up by the time diff (fillShort - flagged).
- The protocol gives users a grace period to reestablish their positions when they fall below the primary liquidation ratio, however in the following situation a user can be liquidated without warning (being flagged).
- A user is also unable to use certain protocol functionality (e.g. transfer his short).

**Tools Used:** 
- Manual Analysis
- Foundry

**Recommendation:** 

The flag needs to be checked in all three exit functions: **`exitShortWallet`**, **`exitShortErcEscrowed`**, and **`exitShort`**, when a short record is fully paid.

Ensure the flag is reset when a user fully pays off their short, so if it was the last short a user will not start of with a healthy position flagged when a new short gets matched at that spot.

```solidity
    if (buyBackAmount == ercDebt) {
            // initial code

	// reset flag here
	short.maybeResetFlag(asset);
	}
```

</details>

<details>
  <summary><a id="l03---xxx"></a>[L03] - Partial filled short does not reset liquidation flag after user gets fully liquidated, meaning healthy position will still be flagged if the rest of the order gets filled</summary>
  
  <br>

**Severity:** Low

**Summary:** 

  - The protocol allows a short order to be partially matched, generating a short record for the matched amount. The unmatched portion of the order can be subsequently filled and added to the short record.
- A short record is flagged if it falls below the primary liquidation ratio set by the protocol, signalling to the user that their position is nearing an unhealthy state. The user can resolve this by modifying the position to improve its health or by paying off the short and exiting the position.
- If a user is unable to get their their position to a healthy state by a certain time they can be liquidated.
- A vulnerability exists where, under specific circumstances, a user’s healthy position is flagged and can be instantly liquidated without warning.

**Vulnerability Details:**

  - Consider the following scenario
    1. User A creates a short order, 50% of which is filled with a bid.
    2. User A’s position falls below the primary liquidation ratio and is flagged by User B.
    3. User A’s position is fully liquidated by User B, with the flag remaining active post liquidation.
    4. The remaining order gets filled at a healthy ratio but remains flagged.
       
<details>
  <summary><b>Click to expand Proof of Concept</b></summary>

  ```solidity
  function testPShortFLiquidatePShort() public {
        skipTimeAndSetEth(2 hours, 4000 ether);

        // Create short
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT * 2, sender);

        // create bid half of short
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);

         //get short 
        STypes.ShortRecord memory shortBeforeFlag =
            getShortRecord(sender, Constants.SHORT_STARTING_ID);

        //check short flag
        assertEq(shortBeforeFlag.flaggerId, 0);

        // fall in price
        skipTimeAndSetEth(2 hours, 2000 ether);

        // flag short
        vm.prank(extra);
        diamond.flagShort(asset, sender, Constants.SHORT_STARTING_ID, Constants.HEAD);

        // skip user grace period
        skipTimeAndSetEth(12 hours, 2000 ether);

        //get short
        STypes.ShortRecord memory shortAfterFlag =
            getShortRecord(sender, Constants.SHORT_STARTING_ID);
        //check short flag
        assertGt(shortAfterFlag.flaggerId, 0);

        // liquidate short
        vm.prank(extra);
        diamond.liquidate(
            asset, sender, Constants.SHORT_STARTING_ID, shortHintArrayStorage
        );

        //get short
        STypes.ShortRecord memory shortAfterExit =
            getShortRecord(sender, Constants.SHORT_STARTING_ID);
        //check short flag
        assertGt(shortAfterExit.flaggerId, 0);

        //price recover back to initial
        skipTimeAndSetEth(2 hours, 4000 ether);

        // rest of the short order gets filled
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
        //get short
        STypes.ShortRecord memory shortAfterMatch =
            getShortRecord(sender, Constants.SHORT_STARTING_ID);
            
         //check short flag
        assertGt(shortAfterMatch.flaggerId, 0);
    }
```
</details>

**Impact:** 

  - A healthy short is incorrectly flagged.
- If the new short falls below the primary liquidation ratio:
  - It cannot be flagged by another user until updatedAt (when short was filled) plus the reset time is reached.
  - It can be liquidated after updatedAt (when short was filled) plus the firstLiquidationTime till resetLiquidationTime even if it was never flagged.
  - Keep in mind the shorts updatedAt will be updated when the short gets filled so this will push the liquidation times up by the time diff (fillShort - flagged).
- A user is also unable to use certain protocol functionality (e.g. transfer the short).

**Tools Used:** 

- Manual Analysis
- Foundry

**Recommendation:** 

- The liquidation process must reset the flag in full liquidations to ensure that users don’t start off with healthy positions flagged when the unmatched portion gets filled.

```solidity
if (m.short.ercDebt == m.ercDebtMatched) {
            // Full liquidation
            LibShortRecord.disburseCollateral(
                m.asset,
                m.shorter,
                m.short.collateral,
                m.short.zethYieldRate,
                m.short.updatedAt
            );
            LibShortRecord.deleteShortRecord(m.asset, m.shorter, m.short.id);
            if (!m.loseCollateral) {
                m.short.collateral -= decreaseCol;
                s.vaultUser[m.vault][m.shorter].ethEscrowed += m.short.collateral;
                s.vaultUser[m.vault][address(this)].ethEscrowed -= m.short.collateral;

			// reset flag here
			short.resetFlag()
            }
```

</details>

<details>
  <summary><a id="l04---xxx"></a>[L04] - XXX</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="l05---xxx"></a>[L05] - XXX</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="l06---xxx"></a>[L06] - XXX</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="l07---xxx"></a>[L07] - XXX</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="l08---xxx"></a>[L08] - XXX</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

<details>
  <summary><a id="l09---xxx"></a>[L09] - XXX</summary>
  
  <br>

  **Severity:** Low

  **Summary:** 

  **Vulnerability Details:** 

  **Impact:** 

  **Tools Used:** 

  **Recommendation:** 

</details>

---
