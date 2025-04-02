# 🔍 Base calculation in Leverager::isLiquidateable() is incorrect as the max leverage may be smaller

## 📌 Summary  

    In the calculation for the base amount in Leverager::isLiquidateable(), the max leverage from the lending pool is not taken into account. This means that the base amount is underestimated, leading to users not being liquidated when they should be. This can lead to losses for the protocol and higher bad debt risk.

## ❌ Why I Missed It  

    I didn't take into account that there are 2 max leverage values, one from the lending pool and one from the Leverager. I thought that the max leverage was always the same in both cases, but it is not. 

## ✅ How I Can Avoid Missing This in the Future  

    I should be more careful with the max leverage values and check if they are the same in all cases. I should also check if there are any other variables that can affect the calculation of the max leverage.

## 🔎 The Finding

Base calculation in Leverager::isLiquidateable() is incorrect as the max leverage may be smaller
Summary
Base calculation in Leverager::isLiquidateable() is:
uint256 base = owedAmount * 1e18 / (vp.maxTimesLeverage - 1e18);.
However, the max leverage may not actually be vp.maxTimesLeverage, but be maxLevTimes from the lending pool. In this case, the base amount would be incorrectly underestimated, leading to users not being liquidated when they should for the protocol's loss (loss of profit and higher bad debt risk).

Root Cause
In Leverager:408, the max leverage from the lending pool is not taken into account.

Internal Pre-conditions
maxLevTimes from the lending pool < vp.maxTimesLeverage.

External Pre-conditions
None.

Attack Path
User has a position that should be liquidated but isn't due to the rhs of the is liquidatable check.
Impact
Protocols takes losses and risks bad debt creation.

PoC
If maxLevTimes < vp.maxTimesLeverage, it means the base calculation would have in the divisor a bigger number than it should, so base will be smaller. As base is smaller, the collateral of the user can decrease more without the user being liquidated.

Mitigation
Compare the 2 max leverage values and use the smallest, which is the actual maximum leverage allowed in the Leverager.