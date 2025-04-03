# 🔍 Finding Title

## 📌 Summary  
_What was the issue?_  
    Theres a problem with pools with odd tick spacing the protocol doesn't calculate for rounding errors and sets wrong wide range for the positon.

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

    Didn't think to check if it works with wrong ticks

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

    Check calculations more in depth and get more familiar with uniswapV3

## 🔎 The Finding

Incorrect _setMainTicks Calculations Causing Shifted Main Position After Rebalance
Summary
In the _setMainTicks function Strategy.sol:236 (https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/Strategy.sol#L236), the calculation logic is incorrect compared to the documentation. This leads to an unintended shift in the main position when rebalancing, particularly for pools with odd tickSpacing values (e.g., 1 or 5).

Root Cause
The calculation for isLowerSided does not account for rounding errors, causing the boolean value to evaluate as false in some cases. As a result, the main position is incorrectly shifted.
Consider the USDC-USDT-0.01% Uniswap V3 pool, where the spot tick is -1. The current _setMainTicks logic produces a new main position with ticks [-2, 2], while the correctly centered position should be [-3, 1].

Internal Pre-conditions
The strategy must be deployed for a pool with an odd tickSpacing (1 or 5).
External Pre-conditions
the spot tick needs to satisfy the following condition:
spotTick
=
k
×
tickSpacing
+
(
tickSpacing
2
)
+
1

where k is an integer.

Example 1: spotTick = -1, tickSpacing = 1
Example 2: spotTick = 2, tickSpacing = 5
Attack Path
Rebalancer rebalances the Strategy.
Impact
The main position shifts to the right by tickSpacing, which is suboptimal and contradicts the intended behavior described in the documentation:

"In order to utilize the most out of the funds within it, 2 positions are made - 1 wider which aims to be 50:50 (or as close as it can get, depending on pool’s tickspacing). "

PoC
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {Vault} from "../src/Vault.sol";
import {IVault} from "../src/interfaces/IVault.sol";
import {IStrategy} from "../src/interfaces/IStrategy.sol";
import {IUniswapV3Pool} from "../src/interfaces/IUniswapV3Pool.sol";
import {IERC20, SafeERC20} from "@openzeppelin/token/ERC20/utils/SafeERC20.sol";
import {ERC20} from "@openzeppelin/token/ERC20/ERC20.sol";
import {Strategy} from "../src/Strategy.sol";
import {IMainnetRouter} from "../src/interfaces/IMainnetRouter.sol";
import {LiquidityAmounts} from "../src/libraries/LiquidityAmounts.sol";
import {TickMath} from "../src/libraries/TickMath.sol";

interface ISwapRouter {
    struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }

    /// @notice Swaps `amountIn` of one token for as much as possible of another token
    /// @param params The parameters necessary for the swap, encoded as `ExactInputSingleParams` in calldata
    /// @return amountOut The amount of the received token
    function exactInputSingle(ExactInputSingleParams calldata params) external payable returns (uint256 amountOut);
}

contract Unit is Test {
    using SafeERC20 for IERC20;

    address public constant UNISWAP_V3_SWAP_ROUTER = 0xE592427A0AEce92De3Edee1F18E0157C05861564;

    address vault;
    address strategy;
    address rebalancer = vm.createWallet("rebalancer").addr;
    address feeRecipient = vm.createWallet("feeRecipient").addr;
    address depositor = vm.createWallet("depositor").addr;
    address owner = vm.createWallet("owner").addr;
    address attacker = vm.createWallet("attacker").addr;

    IUniswapV3Pool pool = IUniswapV3Pool(0x3416cF6C708Da44DB2624D63ea0AAef7113527C6); // USDC-USDT-0.01%
    ERC20 token0 = ERC20(pool.token0());
    ERC20 token1 = ERC20(pool.token1());

    function testSetMainTicks() external {
        // deploy
        vm.startPrank(owner);
        vault = address(new Vault(address(token0), address(token1)));
        strategy = address(new Strategy(address(pool), address(vault), rebalancer, feeRecipient));
        IVault(vault).setStrategy(strategy);
        vm.stopPrank();

        (, int24 tick,,,,,) = pool.slot0();
        assertEq(-1, tick);
        assertEq(-2, Strategy(strategy).getMainPosition().tickLower);
        assertEq(2, Strategy(strategy).getMainPosition().tickUpper);
    }
}

/*
    HOW TO RUN:
        forge test -vvv --fork-url $MAINNET_RPC --fork-block-number 21917372 --match-test testSetMainTicks

    LOGS:
        [PASS] testSetMainTicks() (gas: 7797199)
        Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.93ms (1.87ms CPU time)
*/
Mitigation
Replace the current isLowerSided calculation in Strategy.sol:236 with:

bool isLowerSided = modulo * 2 < tickSpacing;
