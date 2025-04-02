# 🔍 Integer overflow in observation index calculation leads to denial of service

## 📌 Summary  

    Theres is integer overflow when adding 2 uint16 numbers and then performing functions even if the stored value is uint256, they are cast by default to uint16.

## ❌ Why I Missed It  

    First I didn't know that if you have 2 numbers their addition result will be the same type.
    Like here: Both observationCardinality and currentIndex are uint16, meaning they can each hold values up to 2^16 - 1 = 65535.
    
    Secondly I didn't know that the observationCardinality can be increased by anyone by default in UniswapV3.

## ✅ How I Can Avoid Missing This in the Future  

    I have to check more the types of the variables doesn't matter if they are stored in a uint256 variable, they are cast first by the function they are used in if there are multiple functions.
    Also didn't know that the observationCardinality can be increased by anyone by default in UniswapV3, so I will check this in the future.
## 🔎 The Finding

https://github.com/sherlock-audit/2025-02-yieldoor-judging/issues/103

Integer overflow in observation index calculation leads to denial of service
Summary
When observationCardinality and currentIndex are large enough, the index calculation in Strategy#checkPoolActivity will revert due to integer overflow, causing a denial of service for critical protocol functions.

Relevant Context
checkPoolActivity is called by:

Strategy#_requirePriceWithinRange
Strategy#rebalance
Strategy#compound
Strategy#addVestingPosition
Leverager#openLeveragedPosition
These functions are essential for protocol operation.

Root Cause
In Strategy#checkPoolActivity, the observation index calculation:
https://github.com/sherlock-audit/2025-02-yieldoor/blob/b5a0f779dce4236b02665606adb610099451a51a/yieldoor/src/Strategy.sol#L320

uint256 index = (observationCardinality + currentIndex - i) % observationCardinality;
will revert when observationCardinality + currentIndex >= 2**16 due to integer overflow in the addition operation, as both variables are uint16.

Attack Path
An attacker increases observationCardinalityNext to 60000 via UniswapV3Pool#increaseObservationCardinalityNext (See the Appendix for calculation of gas cost)
currentIndex increases so that observationCardinality + currentIndex >= 2**16.
The checkPoolActivity function will revert until currentIndex wraps around to 0.
Impact Explanation
High. The integer overflow causes all critical functions that rely on checkPoolActivity to revert, making the contract completely unusable. This includes:

rebalance() - Cannot adjust positions when market conditions change
compound() - Cannot reinvest earned fees
addVestingPosition() - Cannot add new vesting positions
Most critically, when positions become out-of-range, the protocol cannot rebalance them back into profitable ranges. This leads to:

Loss of yield since out-of-range positions earn no fees
Locked funds that cannot be rebalanced back into range
The issue persists until currentIndex wraps around to 0, which could take a significant amount of time depending on pool activity.

Proof of Concept
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

contract PoC is Test {
    function test_PoC() public {
        uint16 observationCardinality = 2**15 + 1;
        uint16 currentIndex = 2**15;
        uint16 i = 1;
    
        uint256 index = (observationCardinality + currentIndex - i) % observationCardinality;
    }
}
Logs

[FAIL: panic: arithmetic underflow or overflow (0x11)] test_PoC() (gas: 433)
Traces:
  [433] PoC::test_PoC()
    └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 332.96µs (74.17µs CPU time)
Recommendation
Cast observationCardinality to uint32

- uint256 index = (observationCardinality + currentIndex - i) % observationCardinality;
+ uint256 index = (uint32(observationCardinality) + currentIndex - i) % observationCardinality;
Appendix
At block 26923358 on Base tx, on ETH/USDC pool:

observationCardinality = 5000
observationCardinalityNext = 5000
Gas Price: 0.002855 Gwei
2230192 gas to increase observationCardinalityNext by 100. Assuming that 1 ETH = 2500 USD. This means it costs 2230192 * 0.002855 / 1e9 = 6.36719816e-06 ETH = 0.0159179954 USD
It costs ((60000 - 5000) / 100) * 0.0159179954 USD = 8.75489747 USD to increase observationCardinalityNext to 60000.
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {IUniswapV3Pool} from "../src/interfaces/IUniswapV3Pool.sol";
import {IUniswapV3Factory} from "../src/interfaces/IUniswapV3Factory.sol";

contract PoC is Test {
    address WETH = 0x4200000000000000000000000000000000000006;
    address USDT = 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913;
    IUniswapV3Factory factory = IUniswapV3Factory(0x33128a8fC17869897dcE68Ed026d694621f6FDfD);

    function test_PoC() public {
        IUniswapV3Pool pool = IUniswapV3Pool(factory.getPool(WETH, USDT, 500));
        (uint160 sqrtPriceX96, int24 tick,, uint16 observationCardinality, uint16 observationCardinalityNext,,) = pool.slot0();
        console.log("observationCardinality", observationCardinality);
        console.log("observationCardinalityNext", observationCardinalityNext);

        uint256 gasStart = gasleft();
        pool.increaseObservationCardinalityNext(observationCardinality + 100);
        uint256 gasEnd = gasleft();
        console.log("Gas cost of increaseObservationCardinalityNext", gasStart - gasEnd);
    }
}
Logs

Logs:
  observationCardinality 5000
  observationCardinalityNext 5000
  Gas cost of increaseObservationCardinalityNext 2230192