# 🔍 Finding Title

## 📌 Summary  
_What was the issue?_  
- Briefly describe the vulnerability.  
- Which function(s) were affected?  
- How severe was it (low, medium, high, critical)?  

   Precision loss in the calculation of interest in the collateral, because we devide by the borrowing index and if the accured interest amount is less then the borrowing index, it will be lost.

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  
- Was I focusing too much on another part of the code?  
- Did I misinterpret how the function worked?  
- Was I relying too much on a tool or automation?  
- Did I assume something that turned out to be incorrect?  
- Did I lack knowledge about this specific type of vulnerability?  

    Didn't think for the loss of precision if the calculation for total borrows.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  
- What specific checks should I add to my audit workflow?  
- Should I use a different tool or technique?  
- Would a different mindset or approach help?  
- Should I review similar issues in past exploits?  

    Check for loss in precissiong when there is devision in calculations.

## 🔎 The Finding

Insufficient Decimal
Summary
The decimal precision of the collateral is insufficient for calculating the interest, especially for wbtc.

Root Cause
In ReserveLogic::_updateIndexes::L156, the totalBorrows may not update or may experience precision loss.
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/ReserveLogic.sol#L156

156:        newTotalBorrows = newBorrowingIndex * (reserve.totalBorrows) / (reserve.borrowingIndex);
To change the totalBorrows, the unaccounted interest must be equal to or greater than 1 wei.
To avoid precision loss, the interest must exceed 10,000 wei.
If the collateral is wbtc, this value is not small.
And the _updateIndexes function could be executed every block(period = 15s), and the unaccounted interest may be less than the above value.

Internal pre-conditions
N/A

External pre-conditions
N/A

Attack Path
An attacker can repeatedly deposit 1001 wei of wbtc and redeem the maximum amount of wbtc (type(uint256).max).
Alternatively, users can interact with the LendingPool frequently

PoC
Let's consider the following scenario.
LendingPool's asset = wbtc, BorrowingRateConfig: (0%, 0%) -> (80%, 20%) -> (90%, 50%) -> (100%, 150%)
totalLiquidityAndBorrows = 0.25 wbtc, totalBorrows = 0.126144 wbtc
currentUtilizationRate = 50.4576%,
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/InterestRateUtils.sol#L28
currentBorrowingRate = 50.4576% * 20% / 80% = 12.6144%
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/InterestRateUtils.sol#L73
rate = 12.6144% = 0.126144e27, currentTimestamp = lastUpdateTimestamp + 15.
ratePerSecond = rate / SECONDS_PER_YEAR = 4e18
basePowerTwo = ratePerSecond * ratePerSecond / PRECISION = 16e9
basePowerThree = 64
secondTerm = exp * expMinusOne * basePowerTwo / 2 = 15 * 14 * 16e9 = 3360e9
thirdTerm = exp * expMinusOne * expMinusTwo * basePowerThree / 6 = 15 * 14 * 13 * 64 / 6 = 29120
CompoundedInterest = (PRECISION) + (ratePerSecond * exp) + (secondTerm) + (thirdTerm) =
= 1e27 + 4e18 * 15 + 3360e9 + 29120 = 1e27 + 60e18 + 3360e9 + 29120 < 1e27 + 60.1e18
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/ReserveLogic.sol#L115
latestBorrowingIndex = reserve.borrowingIndex * calculateCompoundedInterest / 1e27 < reserve.borrowingIndex * (1e27 + 60.1e18) / 1e27
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/ReserveLogic.sol#L156
newTotalBorrows = latestBorrowingIndex * (reserve.totalBorrows) / (reserve.borrowingIndex) <
< (reserve.borrowingIndex * (1e27 + 60.1e18) / 1e27) * (reserve.totalBorrows) / (reserve.borrowingIndex) =
= (1e27 + 60.1e18) / 1e27 * (reserve.totalBorrows) =
= reserve.totalBorrows + 60.1e-9 * 0.126144e8 <
< reserve.totalBorrows + 1;
Therefore, newTotalBorrows = reserve.totalBorrows

Impact
The depositor of the LendingPool may not be able to take any profits at all or may receive reduced profits.

When BorrowingRateConfig := (0%, 0%) -> (80%, 20%) -> (90%, 50%) -> (100%, 150%),

LendingPool's asset = wbtc, totalLiquidityAndBorrows = 0.25 wbtc, totalBorrows = 0.126144 wbtc
If the _updateIndexes function is executed every block, the interest is not collected.
If it is executed every 100 blocks (25 mins), the calculation of interest experiences 1% precision loss.

LendingPool's asset = wbtc, totalLiquidityAndBorrows = 25 wbtc, totalBorrows = 12.6144 wbtc
If the _updateIndexes function is excuted every 100 block (25 mins), the calculation of interest experiences 0.01% precision loss.

LendingPool's asset = usdc, totalLiquidityAndBorrows = 250,000 usdc, totalBorrows = 126,144 usdc
If the _updateIndexes function is excuted every block, the calculation of interest experiences 0.01% precision loss.

Mitigation
Consider increasing the decimal precision of the collateral.