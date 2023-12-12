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
  <summary><a id="h01---xxx"></a>[H01] - XXX</summary>
  
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
