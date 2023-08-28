# CodeHawks - Escrow Contract

## About

This project is meant to enable smart contract auditors (sellers) and smart contract protocols looking for audits (buyers) to connect using a credibly neutral option, with optional arbitration.

---

## Findings Summary

| ID  | Title                            | Severity   |
|-----|----------------------------------|------------|
| [H01](#h01---xxx) | A permanent revert on transfer among any of the participants in the resolveDispute function could lead to all funds being stuck in the contract                              | High       |
| [M01](#m01---xxx) | Fee on transfer tokens will not be supported                              | Medium     |
| [M02](#m02---xxx) | Lack of input validation in newEscrow function                              | Medium     |

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
  <summary><a id="m01---xxx"></a>[M01] - Fee on transfer tokens will not be supported</summary>
  
  <br>

## **Severity:** 

- Medium Risk

## **Relevant GitHub Links:** 

- https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/EscrowFactory.sol#L20

- https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L44

## **Summary:** 

- The current implementation of the escrow contract does not support tokens that take a fee on transfer. This is because the contract checks if the balance of the contract is greater then the specified price parameter, which will not be the case for tokens that deduct a fee on transfer.

- While this is not an issue at the present time, as many widely-used tokens do not charge a fee, it could become a significant constraint in the future. If popular tokens (such as USDT and USDC) decide to implement a transfer fee, the protocol's regular usability would be severely limited. Consequently, the protocol's user base could be greatly reduced.

## **Vulnerability Details:** 

- In the escrow contracts constructor, a check is performed to ensure that the balance of the escrow contract matches or is greater then the specified price. However, for tokens that deduct a fee on transfer, the balance will be less than the price, causing the function to revert. This effectively means that tokens which employ a transfer fee cannot be used with the current contract implementation.

```solidity
if (tokenContract.balanceOf(address(this)) < price) revert Escrow__MustDeployWithTokenBalance();
```
  
## **Impact:** 

- It will not be possible to use fee on transfer tokens regularly within the protocol

## **Tools Used:** 

- Manual analysis

## **Recommendation:** 

- To address this issue, you could modify the contract to accommodate fee-on-transfer tokens.

- Alternatively, If accommodating fee-on-transfer tokens is not a priority, it would be helpful to clearly document this limitation in the contract comments and any user-facing documentation.

- One work around on the user side would be to have them send extra tokens to the precomputed address before calling the newEscrow function

</details>

---

<details>
  <summary><a id="m02---xxx"></a>[M02] - Lack of input validation in newEscrow function</summary>
  
  <br>

## **Severity:** 

- Medium Risk

## **Relevant GitHub Links:** 

- https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/EscrowFactory.sol#L20

## **Summary:** 

- The newEscrow function lacks input validation checks

	- The arbiter address should be validated to ensure it is not the same as the buyer or seller address. The arbiter is expected to be an impartial, trusted actor who can resolve disputes between the buyer and seller. If the arbiter is also the buyer or seller, this impartiality is compromised.

## **Vulnerability Details:** 

- If the arbiter is also the buyer or seller, it could lead to disputes being resolved unfairly. This is contrary to the intended role of the arbiter as an impartial third party.
  
## **Impact:** 

- The lack of these input validations could lead to disputes being unfairly resolved

## **Tools Used:** 

- Manual analysis

## **Recommendation:** 

- To mitigate these issues, consider adding the following validation checks in the newEscrow function:

```solidity
function newEscrow(
    uint256 price,
    IERC20 tokenContract,
    address seller,
    address arbiter,
    uint256 arbiterFee,
    bytes32 salt
) external returns (IEscrow) {
    require(arbiter != buyer && arbiter != seller, "Arbiter must be different from buyer and seller");
    
    // rest of the function code
}
```
</details>

---

</details>
