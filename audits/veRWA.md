# veRWA - Voting-escrow

---

## About

The contracts implement a voting-escrow incentivization model for Canto RWA (Real World Assets) similar to veCRV with its liquidity gauge. Users can lock up CANTO (for five years) in the VotingEscrow contract to get veCANTO. They can then vote within GaugeController for different lending markets that are white-listed by governance. Users that provide liquidity within these lending markets can claim CANTO (that is provided by CANTO governance) from LendingLedger according to their share.

---

## Findings Summary

| ID  | Title                            | Severity   |
|-----|----------------------------------|------------|
| [H01](#h01---xxx) | Retained Voting Power After Undelegation                              | High       |
| [H02](#h02---xxx) | Removal of a Gauge Locks User Voting Power                              | High       |
| [QA Report](#QA---xxx) | Inflated Balance After Reward Token Reintroduction                              | Grade: A   |

---

## Detailed Findings

### High Findings

<details>
  <summary><a id="h01---xxx"></a>[H01] - Retained Voting Power After Undelegation</summary>
  
  <br>

## **Severity:** 
  
High Risk

## **Summary:** 

When tokens are delegated from one user (User A) to another (User B), the voting power is correctly combined. However, an inconsistency arises when the tokens are undelegated. If User B does not cast another vote after undelegation, they retain the combined voting power. This results in an inflated voting power for User B, which can skew the outcomes of votes. The sequence of events is as follows:

- User A delegates to User B.
- User B votes for Gauge A using the combined voting power.
- In the next epoch, User A undelegates from User B.
- User A votes for Gauge B.
- Despite the undelegation, User B's voting power remains inflated unless they cast another vote.

## **Proof Of Concept:** 

The test case below demonstrates this behaviour:

```solidity
function testDelegateUndelegate() public {
        vm.startPrank(gov);
        gc.add_gauge(gauge1);
        gc.add_gauge(gauge2);
        gc.change_gauge_weight(gauge1, 100);
        gc.change_gauge_weight(gauge2, 100);
        vm.stopPrank();

        vm.deal(user1, 1 ether);
        vm.deal(user2, 1 ether);

        // user 1 create lock
        vm.prank(user1);
        ve.createLock{value: 1 ether}(1 ether);

        // user 2 create lock
        vm.prank(user2);
        ve.createLock{value: 1 ether}(1 ether);

        // user 1 delegate to user 2
        vm.prank(user1);
        ve.delegate(user2);

        // user 2 vote for gauge 2 with user 1's voting power and user 2's voting power
        vm.prank(user2);
        gc.vote_for_gauge_weights(gauge2, 10000);

        // warp to next week
        vm.warp(block.timestamp + 1 weeks);

        console.log("User 1 -> No votes, delegates to User 2");
        console.log("user 2 -> Votes for gauge 2 with User 1's voting power and User 2's voting power");
        // console log divider
        console.log("--------------------------------------------------");

        (uint256 slopeUser1, uint256 powerUser1,) = gc.vote_user_slopes(user1, gauge1);
        console.log("user 1 vote slope", slopeUser1);
        console.log("user 1 vote power", powerUser1);
        // console log divider
        console.log("--------------------------------------------------");

        (uint256 slopeUser2, uint256 powerUser2,) = gc.vote_user_slopes(user2, gauge2);
        console.log("user 2 vote slope", slopeUser2);
        console.log("user 2 vote power", powerUser2);
        // console log divider
        console.log("--------------------------------------------------");

        // check relative weights
        console.log("relative weight g1", gc.gauge_relative_weight_write(gauge1, block.timestamp));
        console.log("relative weight g2", gc.gauge_relative_weight_write(gauge2, block.timestamp));
        // console log divider
        console.log("--------------------------------------------------");
        //check total weights
        console.log("total weight", gc._get_sum());
        // console log divider
        console.log("--------------------------------------------------");

        // user 1 undelegate
        vm.prank(user1);
        ve.delegate(user1);

        //user1 vote for gauge 1
        vm.prank(user1);
        gc.vote_for_gauge_weights(gauge1, 10000);

        gc.checkpoint_gauge(gauge1);
        gc.checkpoint_gauge(gauge2);

        // warp to next week
        vm.warp(block.timestamp + 1 weeks);
        console.log("next week ");
        console.log("User 1 -> Undelegates from User 2, votes for gauge 1");
        console.log("user 2 -> No action");
        // console log divider
        console.log("--------------------------------------------------");

        (uint256 week2slopeUser1, uint256 week2powerUser1,) = gc.vote_user_slopes(user1, gauge1);
        console.log("user 1 vote slope", week2slopeUser1);
        console.log("user 1 vote power", week2powerUser1);
        // console log divider
        console.log("--------------------------------------------------");

        (uint256 week2slopeUser2, uint256 week2powerUser2,) = gc.vote_user_slopes(user2, gauge2);
        console.log("user 2 vote slope", week2slopeUser2);
        console.log("user 2 vote power", week2powerUser2);
        // console log divider
        console.log("--------------------------------------------------");

        // check relative weights
        console.log("relative weight g1", gc.gauge_relative_weight_write(gauge1, block.timestamp));
        console.log("relative weight g2", gc.gauge_relative_weight_write(gauge2, block.timestamp));
        // console log divider
        console.log("--------------------------------------------------");
        //check total weights
        console.log("total weight", gc._get_sum());
    }
```

Analyzing the test case output reveals:

- In the initial stages, User 1 delegates their voting power to User 2, and User 2 votes for gauge 2. At this juncture, the calculations appear accurate.
- However, post-undelegation by User 1 and their subsequent vote for gauge 1, the voting power for gauge 2 remains unchanged from the previous epoch. This effectively means that User 1's voting power gets double-counted.

## **Tools Used:** 

- Manual analysis
- Foundry

## **Recommendation:** 

Reset Votes on delegation: Nullify the delegator's/delegatee’s votes once they delegate their power.

</details>

---

<details>
  <summary><a id="h02---xxx"></a>[H02] - Removal of a Gauge Locks User Voting Power</summary>
  
  <br>

## **Severity:** 
  
High Risk

## **Summary:** 

If a gauge that has previously been voted on by users is removed by governance, the voting power of those users becomes locked. This is due to the inability of users to invoke the voting function to reset their votes for that gauge, as the gauge is marked invalid upon removal.

## **Proof of concept:** 

Below is a test case demonstrating the issue:

```solidity
    function testRemoveGaugeResetVote() public {
        vm.startPrank(gov);
        gc.add_gauge(gauge1);
        gc.add_gauge(gauge2);
        gc.change_gauge_weight(gauge1, 100);
        gc.change_gauge_weight(gauge2, 100);
        vm.stopPrank();

        vm.deal(user1, 1 ether);

        vm.startPrank(user1);
        ve.createLock{value: 1 ether}(1 ether);
        gc.vote_for_gauge_weights(gauge1, 10000);
        vm.stopPrank();

        // warp 4 weeks
        vm.warp(block.timestamp + 4 weeks);
        console.log("week 4");

        // remove gauge
        vm.startPrank(gov);
        gc.remove_gauge(gauge1);
        vm.stopPrank();

        // user1 tries to change vote
        vm.startPrank(user1);
        vm.expectRevert("Invalid gauge address");
        gc.vote_for_gauge_weights(gauge1, 0);
        vm.stopPrank();
    }
```

In the test above:

- Two Gauges are added, and user votes for gauge1.
- Time is advanced by 4 weeks.
- Governance removes gauge1.
- User1 tries to reset their vote for gauge1 to 0, but it fails due to the "Invalid gauge address" check.

## **Tools Used:** 

- Manual analysis
- Foundry

## **Recommendation:** 

- Implement a mechanism to automatically reset the votes for users who have voted on a gauge upon its removal.
- Alternatively, provide a function that allows users to reset their votes for removed gauges without the isValidGauge check.

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
