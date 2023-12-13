# veRWA - Voting-escrow

The audit report will be updated with the findings as soon as the Code4rena team finalizes and releases their report.

---

## About

The contracts implement a voting-escrow incentivization model for Canto RWA (Real World Assets) similar to veCRV with its liquidity gauge. Users can lock up CANTO (for five years) in the VotingEscrow contract to get veCANTO. They can then vote within GaugeController for different lending markets that are white-listed by governance. Users that provide liquidity within these lending markets can claim CANTO (that is provided by CANTO governance) from LendingLedger according to their share.

---

## Findings Summary

| ID  | Title                            | Severity   |
|-----|----------------------------------|------------|
| [H01](#h01---xxx) | XXX                              | High       |
| [H02](#h02---xxx) | XXX                              | High       |
| [QA Report](#QA---xxx) | Inflated Balance After Reward Token Reintroduction                              | Grade: A   |

---

## Detailed Findings

### High Findings

<details>
  <summary><a id="h01---xxx"></a>[H01] - XXX</summary>
  
  <br>

## **Severity:** 
  
- High Risk

## **Relevant GitHub Links:** 

- 

## **Summary:** 

- 

## **Vulnerability Details:** 

- 

```solidity

```

- 
  
```solidity

```
  
## **Impact:** 

- 

## **Tools Used:** 

- 

## **Recommendation:** 

- 

</details>

---

<details>
  <summary><a id="h02---xxx"></a>[H02] - XXX</summary>
  
  <br>

## **Severity:** 
  
- High Risk

## **Relevant GitHub Links:** 

- 

## **Summary:** 

- 

## **Vulnerability Details:** 

- 

```solidity

```

- 
  
```solidity

```
  
## **Impact:** 

- 

## **Tools Used:** 

- 

## **Recommendation:** 

- 

</details>

---

### QA Report

<details>
  <summary><a id="QA---xxx"></a>[QA] - Inflated Balance After Reward Token Reintroduction</summary>
  
  <br>

## **Severity:** 

- QA Report

## **Summary:** 

The functionality exists in the protocol for governance to vote and remove a reward token and later reintroduce it. If this occurs, users might experience inflated balances. This can lead to significant losses for the protocol, as users could exploit this to earn more rewards than they're entitled to.

## **Vulnerability Details:** 

Consider the following sequence of events:

- User A deposits 100 units of token X into the liquidity pool, enabling them to earn rewards from token A.
- Governance decides to delist token A, effectively removing it from the whitelist.
- User A withdraws their 100 units of token X. Since token A is not on the whitelist, the syncLedger function fails to register this withdrawal.
- After a month, token A is reintroduced and added back to the whitelist by governance.
- User A now deposits an additional 10 units of token X. However, the syncLedger function, attempting to bridge the data gap, incorrectly adds this to the old balance.
- This results in User A appearing to have a balance of 110 units of token X, when in reality they only - deposited 10 units.
- Consequently, User A starts to accumulate rewards at a rate based on this inflated balance.

```solidity
function testSyncLedgerWithGaps() public {
        // prepare
        whiteListMarket();
        vm.warp(block.timestamp + WEEK);

        vm.startPrank(lendingMarket);
        int256 deltaStart = 1 ether;
        uint256 epochStart = (block.timestamp / WEEK) * WEEK;
        ledger.sync_ledger(lender, deltaStart);
        vm.stopPrank();

        // remove market from whitelist
        vm.prank(goverance);
        ledger.whiteListLendingMarket(lendingMarket, false);

	// user removes liquidity tokens

        // warp 2 week
        uint256 newTime = block.timestamp + 3 * WEEK;
        vm.warp(newTime - 1 weeks);

        // add market to whitelist
        vm.prank(goverance);
        ledger.whiteListLendingMarket(lendingMarket, true);

        // warp 1 week
        vm.warp(block.timestamp + 1 weeks);

        int256 deltaEnd = 1 ether;
        uint256 epochEnd = (newTime / WEEK) * WEEK;

        uint256 lenderBalanceInitial = ledger.lendingMarketBalances(lendingMarket, lender, epochEnd);
        console.log("epochEnd", lenderBalanceInitial);
        vm.prank(lendingMarket);
        ledger.sync_ledger(lender, deltaEnd);

        // lender balance is forwarded and set
        uint256 lenderBalance = ledger.lendingMarketBalances(lendingMarket, lender, epochEnd);
        assertEq(lenderBalance, uint256(deltaStart) + uint256(deltaEnd));

        // total balance is forwarded and set
        uint256 totalBalance = ledger.lendingMarketTotalBalance(lendingMarket, epochEnd);
        assertEq(totalBalance, uint256(deltaStart) + uint256(deltaEnd));
    }
```

Analyzing the test case, it becomes evident that the user balance is reflective of the user's prior interactions with the token, pre-removal, and subsequent addition. This means any withdrawals made by the user during the interim (between removal and re-addition) are overlooked.

## **Tools Used:** 

- Manual analysis
- Foundry 

## **Recommendation:** 

Whenever a token is reintroduced, ensure that all its associated values are reset to their default states. This should be done irrespective of any previous interactions with that token.

</details>

---
