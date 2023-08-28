# CodeHawks - Escrow Contract

## About

This project is meant to enable smart contract auditors (sellers) and smart contract protocols looking for audits (buyers) to connect using a credibly neutral option, with optional arbitration.

---

## Scope

---

## Findings Summary

| ID  | Title                            | Severity   |
|-----|----------------------------------|------------|
| [H01](#h01---xxx) | A permanent revert on transfer among any of the participants in the resolveDispute function could lead to all funds being stuck in the contract                              | High       |
| [M01](#m01---xxx) | XXX                              | Medium     |
| [M02](#m02---xxx) | XXX                              | Medium     |

---

## Detailed Findings

### High Findings

<details>
  <summary><a id="h01---xxx"></a>[H01] - A permanent revert on transfer among any of the participants in the resolveDispute function could lead to all funds being stuck in the contract</summary>
  
  <br>

## **Severity:** 
  
- High Risk

## **Relevant GitHub Links:** 

- https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L109

## **Summary:** 

- In the resolveDispute function of the Escrow contract, all funds are distributed in the same function among the participants using a push over pull method. One revert could lead to locking all funds in the worst case. To mitigate this, it is recommended to use a pull over push strategy and have a different function for each transfer.

## **Vulnerability Details:** 

- In the resolveDispute function of the Escrow contract, tokens could fail to transfer due to various reasons. For instance, the token of choice could:

  - Implement an admin-controlled address blocklist (e.g., USDC and USDT), which can cause a revert on the resolveDispute function. This could lead to funds getting locked in the contract, or necessitate sending funds to other users first to avoid getting stuck.

  - Have callbacks that allow malicious users to DOS dispute resolutions, as mentioned in the Known Issues section.

- Consider the most adverse scenario:

  - The escrow has moved into the dispute stage, and the arbiter, who has set a fee greater than zero, is either blocked by the token in use or intentionally disrupts the transfer in a DoS attack. This would cause the transfer operation to consistently fail, leading to a situation where the function cannot execute successfully. As a result, all the funds will remain indefinitely locked within the contract.
  
## **Impact:** 

- Impact: High, as essential functionality in the protocol could fail.

- Likelihood: Low, as a specific type of ERC20 token must be used and one of the above conditions must be met.

## **Tools Used:** 

- Manual analysis

## **Recommendation:** 

- To mitigate this issue, consider using a pull over push strategy to send tokens and separate each transfer. This can be implemented as follows:

- Set the final values in the resolveDispute function:

```solidity
//state variables
uint private buyerShare;
uint private sellerShare;
uint private arbiterShare;

function resolveDispute(uint256 buyerAward) external onlyArbiter nonReentrant inState(State.Disputed) {
		tokenBalance = i_tokenContract.balanceOf(address(this));
        // existing code...

        if (buyerAward > 0) {
			buyerShare = buyerAward;
        }

		if (i_arbiterFee > 0) {
            arbiterShare = i_arbiterFee
        }

		uint currTotal = buyerShare + arbiterShare

        if (tokenBalance - currTotal > 0) {
            sellerShare = tokenBalance - currTotal
        }
}
```

- Create individual claim functions for each user:

```solidity
function buyerClaim() external inState(State.resolved) {
		if (buyerShare > 0) {
            i_tokenContract.safeTransfer(i_buyer, buyerShare);
			buyerShare = 0;
        }
}
```

```solidity
function sellerClaim() external inState(State.resolved) {
		if (sellerShare > 0) {
            i_tokenContract.safeTransfer(i_seller, sellerShare);
			sellerShare = 0
        }
}
```

```solidity
function arbiterClaim() external inState(State.resolved) {
		if (arbiterShare > 0) {
            i_tokenContract.safeTransfer(i_arbiter, arbiterShare);
			arbiterShare = 0;
        }
}
```
- These changes will limit the scope of potential issues. If a transfer fails, it will only affect the single user, rather than all users. Additionally, it can help to mitigate the effects of a known issue where malicious users can DOS dispute resolutions.

</details>

---

### Medium Findings

<details>
  <summary><a id="m01---xxx"></a>[M01] - XXX</summary>
  
  <br>

## **Severity:** 

- Medium Risk

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

### Low Findings

<details>
  <summary><a id="l01---xxx"></a>[L01] - XXX</summary>
  
  <br>

## **Severity:** 

- Low Risk

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
