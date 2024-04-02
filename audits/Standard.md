# The Standard - Findings Report

# Table of contents

- ## High Risk Findings
  - ### [H-01. consolidatePendingStakes function could hit the gas limit causing dos](#H-01)
  - ### [H-02. swap fees going to the liquidation pool manager contract will be accounted for as part of the liquidation amount](#H-02)
- ## Medium Risk Findings
  - ### [M-01. Inconsistency in Price Calculation Leading to Inaccurate Collateral Swaps](#M-01)
  - ### [M-02. distributeAssets function doesn’t recover liquidated positions debt leaving the protocol to cover the losses](#M-02)
  - ### [M-03. Users can borrow a large amount of tokens for free when a token is removed from accepted tokens](#M-03)
- ## Low Risk Findings
  - ### [L-01. Inability to Claim Rewards for Removed Tokens in Liquidation Pool contract](#L-01)
  - ### [L-02. Incorrect Fee Calculation in LiquidationPool's Position Function](#L-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 2
- Medium: 3
- Low: 2

# High Risk Findings

## <a id='H-01'></a>H-01. consolidatePendingStakes function could hit the gas limit causing dos

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L119

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L134

## Summary

In the LiquidationPool contract, the **increasePosition** function allows users to open or increase their staking position, setting transactions in a pending state for at least one day. The consolidatePendingStakes function is then employed to finalize these positions. However, there's a potential risk of this function running out of gas, which could lead to a denial of service.

```solidity
function increasePosition(uint256 _tstVal, uint256 _eurosVal) external {
        require(_tstVal > 0 || _eurosVal > 0);
        consolidatePendingStakes();
        ILiquidationPoolManager(manager).distributeFees();
        if (_tstVal > 0) IERC20(TST).safeTransferFrom(msg.sender, address(this), _tstVal);
        if (_eurosVal > 0) IERC20(EUROs).safeTransferFrom(msg.sender, address(this), _eurosVal);
        pendingStakes.push(PendingStake(msg.sender, block.timestamp, _tstVal, _eurosVal));
        addUniqueHolder(msg.sender);
    }
```

The consolidatePendingStakes function processes all eligible positions in the pendingStakes array that have been pending for over 24 hours. The absence of an upper limit on the array's size and the function's mechanism to loop through the entire array could result in gas consumption exceeding the block gas limit in scenarios with a large number of pending stakes.

```solidity
function consolidatePendingStakes() private {
        uint256 deadline = block.timestamp - 1 days;
        for (int256 i = 0; uint256(i) < pendingStakes.length; i++) {
            PendingStake memory _stake = pendingStakes[uint256(i)];
            if (_stake.createdAt < deadline) {
                positions[_stake.holder].holder = _stake.holder;
                positions[_stake.holder].TST += _stake.TST;
                positions[_stake.holder].EUROs += _stake.EUROs;
                deletePendingStake(uint256(i));
                // pause iterating on loop because there has been a deletion. "next" item has same index
                i--;
            }
        }
    }
```

This could potentially lead to consistently failing transactions due to excessive gas requirements, effectively causing a denial of service and hindering the staking process.

## **Impact:**

If the consolidatePendingStakes function is unable to execute due to gas limitations, it would disrupt the normal operation of the LiquidationPool contract. This interruption could prevent the finalization of pending stakes, affecting the overall functionality of the staking mechanism.

## **Tools Used:**

Manual analysis

## **Recommendation:**

Modify the consolidatePendingStakes function to incorporate an early termination feature. Since the pendingStakes array is ordered from oldest to newest, the function should terminate as soon as it encounters the first stake that does not meet the 24-hour threshold. This approach will ensure efficient processing by only iterating through relevant stakes, significantly reducing the function's gas consumption.

## <a id='H-02'></a>H-02. swap fees going to the liquidation pool manager contract will be accounted for as part of the liquidation amount

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L214

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L196

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L190

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPoolManager.sol#L59

## **Summary:**

In the protocol, users can create Smart Vaults to deposit collateral and borrow EURO stablecoins. They also have the option to swap collateral types, incurring a swap fee that is transferred to the LiquidationPoolManager contract. A issue arises because these swap fees, meant for the protocol, are incorrectly included in the liquidation amounts.

## **Vulnerability Details:**

The SmartVaultV3 contract's swap function charges a fee for each collateral swap. This fee is forwarded to the LiquidationPoolManager contract via either the **executeNativeSwapAndFee** or **executeERC20SwapAndFee** function.

```solidity
function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
        uint256 swapFee =
            _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        ...
        inToken == ISmartVaultManagerV3(manager).weth()
            ? executeNativeSwapAndFee(params, swapFee)
            : executeERC20SwapAndFee(params, swapFee);
    }
```

These functions transfer the collected swap fee to the LiquidationPoolManager:

```solidity
function executeNativeSwapAndFee(ISwapRouter.ExactInputSingleParams memory _params, uint256 _swapFee) private {
        (bool sent,) = payable(ISmartVaultManagerV3(manager).protocol()).call{value: _swapFee}("");
        require(sent, "err-swap-fee-native");
        ...
    }

function executeERC20SwapAndFee(ISwapRouter.ExactInputSingleParams memory _params, uint256 _swapFee) private {
        IERC20(_params.tokenIn).safeTransfer(ISmartVaultManagerV3(manager).protocol(), _swapFee);
        ...
    }
```

The issue emerges as swap fees accumulate in the LiquidationPoolManager contract and are erroneously considered as part of the liquidation assets.

Specifically, the **runLiquidation** function within the LiquidationPoolManager contract, triggered during a liquidation event, assesses the liquidated assets' value using the **balanceOf** function. This will include the swap fees in its calculation, thus conflating them with the liquidation assets.

```solidity
function runLiquidation(uint256 _tokenId) external {
        ...
        ITokenManager.Token[] memory tokens = ITokenManager(manager.tokenManager()).getAcceptedTokens();
        ILiquidationPoolManager.Asset[] memory assets = new ILiquidationPoolManager.Asset[](tokens.length);
        uint256 ethBalance;
        for (uint256 i = 0; i < tokens.length; i++) {
            ITokenManager.Token memory token = tokens[i];
            if (token.addr == address(0)) {
                ethBalance = address(this).balance;
                if (ethBalance > 0) assets[i] = ILiquidationPoolManager.Asset(token, ethBalance);
            } else {
                IERC20 ierc20 = IERC20(token.addr);
                uint256 erc20balance = ierc20.balanceOf(address(this));
                if (erc20balance > 0) {
                    assets[i] = ILiquidationPoolManager.Asset(token, erc20balance);
                    ierc20.approve(pool, erc20balance);
                }
            }
        }
        LiquidationPool(pool).distributeAssets{value: ethBalance}(assets, manager.collateralRate(), manager.HUNDRED_PC());
        ...
    }
```

As a result, swap fees are inadvertently treated as part of the liquidated assets and are distributed during the liquidation process, leading to an incorrect allocation of funds.

## Impact:

This miscalculation leads to the unintended distribution of swap fees as part of liquidated assets causing losses to the protocol. Furthermore, an incorrectly inflated liquidation amount can disrupt the protocol's accounting balance and potentially give rise to further complications.

## **Tools Used:**

Manual analysis

## **Recommendation:**

One solution is to transfer these fees to a different address, ensuring they are not mistakenly included in liquidation distributions.

# Medium Risk Findings

## <a id='M-01'></a>M-01. Inconsistency in Price Calculation Leading to Inaccurate Collateral Swaps

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L67

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L214

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L206

## **Summary:**

Users can create Smart Vaults to deposit collateral and borrow EURO stablecoins. They also have the option to swap collateral types, as long as the return amount from the swap keeps the position at a healthy ratio. This is enforced by the calculateMinimumAmountOut function, however there is a problem in the way it retrieves its prices which could lead inaccurate amounts.

## **Vulnerability Details:**

The **calculateMinimumAmountOut** function calculates **collateralValueMinusSwapValue** with the following formula:

```solidity
 uint256 collateralValueMinusSwapValue =
            euroCollateral() - calculator.tokenToEur(getToken(_inTokenSymbol), _amount);
```

To calculate this, the function initially assesses the total value of the user's collateral in euros. This is done using the **tokenToEurAvg** method from the calculator contract, which calculates an average based on the last four prices of the token:

```solidity
function euroCollateral() private view returns (uint256 euros) {
        ITokenManager.Token[] memory acceptedTokens = getTokenManager().getAcceptedTokens();
        for (uint256 i = 0; i < acceptedTokens.length; i++) {
            ITokenManager.Token memory token = acceptedTokens[i];
            euros += calculator.tokenToEurAvg(token, getAssetBalance(token.symbol, token.addr));
        }
    }
```

However, when subtracting the value of the token being swapped from the total collateral, the formula employs **tokenToEur**, which provides the latest price of the token, rather than an average. This results in a potential mismatch in the valuation method used within the same calculation:

## Scenario illustration:

- User A swaps their entire collateral of 1 ETH for USD.
- The latest ETH price experiences a sharp decline (e.g., 200, 500, 501, 502).
- The average price (**tokenToEurAvg**) would be significantly higher than the latest price (**tokenToEur** at 200).

The potential inflation or deflation of the **calculateMinimumAmountOut** value due to inconsistent pricing methods can significantly impact the swap process. This value, used as **amountOutMinimum** in swaps, can lead to inaccuracies in collateral value assessments. In the worst case, it could lead to the user's position becoming undercollateralized, potentially triggering an unwarranted liquidation.

```solidity
function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
        ...
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            ...
            amountOutMinimum: minimumAmountOut,
            ...
    }
```

## Impact

This inconsistency in price calculations can result in an inaccurate evaluation of swaps, affecting the collateral ratios and potentially leading to liquidations.

## **Tools Used:**

Manual analysis

## **Recommendation:**

The protocol should standardize the price retrieval method in the **calculateMinimumAmountOut** function to ensure consistency. Either both components of the formula should use the average price or the latest price to maintain accuracy in collateral swap calculations.

## <a id='M-02'></a>M-02. distributeAssets function doesn’t recover liquidated positions debt leaving the protocol to cover the losses

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L219

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L220

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L75

## **Summary:**

The protocol allows users to stake TST and EUROs in the Liquidation Pool, offering rewards derived from borrowing fees and liquidations. However, an issue arises in the liquidation process. When liquidated positions are sold to stakers at a discounted rate, based on their stake proportion, the proceeds from these sales fall short of covering the original debts of the positions. This results in losses for the protocol.

## **Vulnerability Details:**

The **distributeAssets** function, used in liquidations, disperses assets from liquidated positions to stakers based on their stake's weight and at a discounted rate.

The user's share from the liquidated position is calculated as follows:

```solidity
uint256 _portion = asset.amount * _positionStake / stakeTotal;
```

The cost in euros that a user pays for this portion is calculated with a discount based on the initial collateral rate:

```solidity
 uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
                            * _hundredPC / _collateralRate;
```

However, this discount rate coincides with the rate at which borrowers are liquidated at. This means that when borrowers are liquidated under this rate, the discounted price offered to stakers results in the protocol recovering less than the original debt amount, incurring losses.

```solidity
function maxMintable() private view returns (uint256) {
        return euroCollateral() * ISmartVaultManagerV3(manager).HUNDRED_PC()
            / ISmartVaultManagerV3(manager).collateralRate();
    }
```

## Proof Of Concept

Consider this scenario:

User A deposits 110 USD of collateral and borrows 100 EUROs at a collateral rate of 110%.

- maxMintable = euroCollateral x HUNDRED_PC / collateralRate = 110 x 100000 / 110000 = 100USD

The collateral value drops to 105 USD, triggering liquidation.

The liquidated assets are distributed as follows (assuming 1 staker):

- \_portion = 105 USD x 1 / 1 = 105 USD
- costInEuros = 105 USD x 1 / 1 x 100000 / 110000 = 95 EUROs

In this example, the liquidated assets are sold for 95 EUROs, less than the initial debt of 100 EUROs, causing the protocol to incur a loss of 5 EUROs.

## Impact:

The protocol faces consistent losses in its liquidation process, as it consistently fails to recover the full amount of the initial debt. These losses accumulate with each liquidation.

## **Tools Used:**

- Manual analysis

## **Recommendation:**

The **distributeAssets** function needs to be restructured to ensure that the protocol can adequately cover the debt from liquidated positions while still providing stakers with a discount. This could be achieved by adjusting the discount rate applied in the **distributeAssets** function, ensuring a more balanced and financially sustainable liquidation process.

## <a id='M-03'></a>M-03. Users can borrow a large amount of tokens for free when a token is removed from accepted tokens

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L119-L128

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L102-L104

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L67-L73

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L113-L117

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L170-L177

## **Summary:**

The protocol maintains an **accepted tokens** array to identify eligible collateral tokens. However, an issue arises when a token, previously accepted and used as collateral, is removed from this list. The SmartVaultV3 contract fails to account for such tokens in its collateral calculations and transfers post-removal. This allows users to obtain free tokens, exploiting the system.

## **Vulnerability Details:**

The **SmartVaultV3** contract triggers a liquidation for undercollateralized positions based on the **euroCollateral()** function, which calculates collateral value using the tokens returned by the **getAcceptedTokens()** function.

```solidity
function euroCollateral() private view returns (uint256 euros) {
        ITokenManager.Token[] memory acceptedTokens = getTokenManager().getAcceptedTokens();
        for (uint256 i = 0; i < acceptedTokens.length; i++) {
            ITokenManager.Token memory token = acceptedTokens[i];
            euros += calculator.tokenToEurAvg(token, getAssetBalance(token.symbol, token.addr));
        }
    }
```

During a liquidation, all collateral tokens are sent to the **liquidationPoolManager** contract. Both the collateral valuation and transfer processes rely on the **accepted tokens** array. Thus, if a token is removed from this array, any position holding it becomes undercollateralized yet exempt from liquidation, creating a loophole.

```solidity
function liquidate() external onlyVaultManager {
        require(undercollateralised(), "err-not-liquidatable");
        liquidated = true;
        minted = 0;
        liquidateNative();
        ITokenManager.Token[] memory tokens = getTokenManager().getAcceptedTokens();
        for (uint256 i = 0; i < tokens.length; i++) {
            if (tokens[i].symbol != NATIVE) liquidateERC20(IERC20(tokens[i].addr));
        }
    }
```

Additionally, the **removeAsset** function allows users to withdraw tokens outside the eligible tokens list, further exploiting the problem.

```solidity
function removeAsset(address _tokenAddr, uint256 _amount, address _to) external onlyOwner {
        ITokenManager.Token memory token = getTokenManager().getTokenIfExists(_tokenAddr);
        if (token.addr == _tokenAddr) require(canRemoveCollateral(token, _amount), UNDER_COLL);
        IERC20(_tokenAddr).safeTransfer(_to, _amount);
        emit AssetRemoved(_tokenAddr, _amount, _to);
    }
```

## **Impact:**

This could lead to substantial losses for the protocol, as it prevents the full liquidation of positions containing the removed token, leaving associated debts unpaid.

Moreover, informed users can exploit this flaw in the following way:

- User A, anticipating the removal of Token X from the list of accepted collateral, mints a substantial amount of tokens using Token X as collateral.
- The protocol proceeds to remove Token X from the list of accepted collateral.
- Following this removal, User A's position becomes undercollateralized. However, due to the nature of the vulnerability, this under collateralization does not trigger the usual liquidation process, leaving User A's collateral unaffected.
- User A then calls the **removeAsset** function to withdraw all of their Token X collateral from the position.

Note: A flash loan can be used to sandwich the removal call to maximize the exploit.

## Proof Of Concept

```jsx
it("User front run removal of token to borrow a large amount of tokens for free", async () => {
  // set up
  const SUSD18 = await (
    await ethers.getContractFactory("ERC20Mock")
  ).deploy("sUSD18", "SUSD18", 18);
  const ClUsdUsd = await (
    await ethers.getContractFactory("ChainlinkMock")
  ).deploy("USD / USD");
  await ClUsdUsd.setPrice(100000000);
  await TokenManager.addAcceptedToken(SUSD18.address, ClUsdUsd.address);
  const SUSD18value = ethers.utils.parseEther("1000");

  // user flashloans and transfers 1000 SUSD18 to vault
  await SUSD18.mint(Vault.address, SUSD18value);
  // minting value
  const mintedValue = ethers.utils.parseEther("500");
  // minting fee
  const mintingFee = mintedValue.mul(PROTOCOL_FEE_RATE).div(HUNDRED_PC);
  // user mints 500 EUROs
  await expect(Vault.connect(user).mint(user.address, mintedValue)).not.to.be
    .reverted;
  // token manager removes asset from accepted tokens
  await TokenManager.removeAcceptedToken(
    "0x5355534431380000000000000000000000000000000000000000000000000000" // SUSD18
  );
  // user removes all collateral from vault (pay back flash loan) bypassing check because token no longer part of accepted tokens
  await expect(
    Vault.connect(user).removeAsset(SUSD18.address, SUSD18value, user.address)
  ).not.to.be.reverted;
  // check balance of vault collateral
  expect(await SUSD18.balanceOf(Vault.address)).to.equal(0);
  // check amount minted
  const { minted } = await Vault.status();
  // minted is equal to mintedValue + mintingFee
  expect(minted).to.equal(mintedValue.add(mintingFee));
});
```

## **Tools Used:**

- Manual analysis
- Hardhat

## **Recommendation:**

When considering the removal of a token, the protocol should assess its impact on existing positions that are utilizing it. One approach could be to disallow the use of the token for collateral in new borrowings while maintaining its validity for existing positions to ensure they remain adequately collateralized. This token can then be gradually phased out.

# Low Risk Findings

## <a id='L-01'></a>L-01. Inability to Claim Rewards for Removed Tokens in Liquidation Pool contract

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L164

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L205

## **Summary:**

The protocol permits users to stake TST and/or EUROs in the Liquidation Pool, accruing rewards from borrowing fees and vault liquidations. These rewards are claimable by stakers at their discretion. However, an issue arises when a token, once accepted and used in liquidations, is subsequently removed from the protocol. This removal prevents users from claiming their pending rewards in the removed token.

## **Vulnerability Details:**

The **claimRewards** function enables users to claim their liquidation rewards, which are available for claiming anytime. This function relies on the **getAcceptedTokens** method to identify tokens eligible for reward distribution.

```solidity
function claimRewards() external {
        ITokenManager.Token[] memory _tokens = ITokenManager(tokenManager).getAcceptedTokens();
        for (uint256 i = 0; i < _tokens.length; i++) {
            ITokenManager.Token memory _token = _tokens[i];
            uint256 _rewardAmount = rewards[abi.encodePacked(msg.sender, _token.symbol)];
            if (_rewardAmount > 0) {
                delete rewards[abi.encodePacked(msg.sender, _token.symbol)];
                if (_token.addr == address(0)) {
                    (bool _sent,) = payable(msg.sender).call{value: _rewardAmount}("");
                    require(_sent);
                } else {
                    IERC20(_token.addr).transfer(msg.sender, _rewardAmount);
                }
            }

        }
    }
```

The issue arises when the protocol’s list of accepted tokens is modified, particularly when tokens are removed. Any rewards pending in the removed token becomes unclaimable, as the **claimRewards** function ceases to recognize them as part of the rewardable assets.

## Impact

Users will be unable to claim rewards that they have earned from liquidations involving tokens that have been removed from the accepted list.

## **Tools Used:**

Manual analysis

## **Recommendation:**

Modify the reward claiming process to include both current and previously accepted tokens. This adjustment will ensure that users can always claim their rewards, irrespective of any changes in the accepted tokens list.

## <a id='L-02'></a>L-02. Incorrect Fee Calculation in LiquidationPool's Position Function

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L83

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPoolManager.sol#L33

## **Summary:**

The **position** function in the LiquidationPool contract, which displays a holder's position (EUROs, TST, and rewards), currently yields inaccurate data due to a flaw in its implementation. This function is essential for individual holders and frontends to accurately display position data.

## **Vulnerability Details:**

The issue lies in how the **position** function calculates the fees a user is entitled to from the **distributeFees** function, in the scenario where the user's TST balance is greater than zero. It erroneously uses the total EUROs balance from the manager contract for this calculation.

```solidity
function position(address _holder) external view returns(Position memory _position, Reward[] memory _rewards) {
        _position = positions[_holder];
        (uint256 _pendingTST, uint256 _pendingEUROs) = holderPendingStakes(_holder);
        _position.EUROs += _pendingEUROs;
        _position.TST += _pendingTST;
        if (_position.TST > 0) _position.EUROs += IERC20(EUROs).balanceOf(manager) * _position.TST / getTstTotal();
        _rewards = findRewards(_holder);
    }
```

However, the **distributeFees** function only allocates a fraction of this balance to the pool, specifically governed by the **poolFeePercentage**. As a result, the fee portion calculated in the **position** function becomes significantly overestimated.

```solidity
function distributeFees() public {
        IERC20 eurosToken = IERC20(EUROs);
        uint256 _feesForPool = eurosToken.balanceOf(address(this)) * poolFeePercentage / HUNDRED_PC;
        if (_feesForPool > 0) {
            eurosToken.approve(pool, _feesForPool);
            LiquidationPool(pool).distributeFees(_feesForPool);
        }
        eurosToken.transfer(protocol, eurosToken.balanceOf(address(this)));
    }
```

## **Impact:**

This miscalculation in the **position** function leads to incorrect reporting of holders' positions. Users and frontends relying on this function for position information will receive inflated values, potentially causing confusion and mismanagement of assets.

## **Tools Used:**

Manual analysis

## **Recommendation:**

The protocol should adjust the **position** function to accurately reflect the actual fee distribution. This involves modifying the calculation to consider only the portion of EUROs allocated as fees, in line with the **poolFeePercentage**, rather than the total EUROs balance.
