# 🔍 Finding Title

## 📌 Summary  
_What was the issue?_  
- Briefly describe the vulnerability.  
- Which function(s) were affected?  
- How severe was it (low, medium, high, critical)?  

    When a vesting position is created and pool price moves outside of range the rebalnce isn't done correctly. Which will call the protocol main functions to revert

    collectPositionFees(vestPosition.tickLower, mainPosition.tickUpper);

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

    I saw this but thought this is a correct part of the protocol.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

    When I see inconsitencyes always double check.
    
## 🔎 The Finding

System Lockup Due to Invalid Ticks in Vesting Positions.
Summary
System Insolvency Due to Invalid Ticks in Vesting Position
The improper handling of vested positions in the Strategy::collectFees function causes most system functionalities to fail, as collectFees may revert with an "NP" error.

Root Cause
Strategy.sol:167 - must be collectPositionFees(vestPosition.tickLower, vestPosition.tickUpper);

Internal Pre-conditions
Owner needs to call addVestingPosition with the specific _vestDuration
Rebalancer needs to call rebalance
External Pre-conditions
After a vesting position is added — and before rebalancing — the pool's price must move outside the boundaries of the mainPosition.
Attack Path
Owner must call addVestingPosition with the specified _vestDuration.
After this call — and before both the next step and the vesting position's expiration — the pool’s price must move outside the bounds of the mainPosition.
Finally, the Rebalancer must invoke the rebalance function before the vesting position expires.
Impact
Due to invalid parameters passed on line Strategy.sol:167 (https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/Strategy.sol#L167), the Strategy::collectFees() function reverts with an "NP" error, effectively locking all user funds in the system.

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

    address public constant UNISWAP_V3_FACTORY = 0x1F98431c8aD98523631AE4a59f267346ea31F984;
    address public constant UNISWAP_V3_SWAP_ROUTER = 0xE592427A0AEce92De3Edee1F18E0157C05861564;

    address vault;
    address strategy;
    address rebalancer = vm.createWallet("rebalancer").addr;
    address feeRecipient = vm.createWallet("feeRecipient").addr;
    address depositor = vm.createWallet("depositor").addr;
    address owner = vm.createWallet("owner").addr;
    address attacker = vm.createWallet("attacker").addr;

    // HOW TO RUN: forge test -vvv --fork-url $MAINNET_RPC --fork-block-number 21917372 --match-test testVestPosition
    function testVestPosition() external {
        IUniswapV3Pool pool = IUniswapV3Pool(0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640);
        ERC20 token0 = ERC20(pool.token0());
        ERC20 token1 = ERC20(pool.token1());

        // deploy
        {
            vm.startPrank(owner);
            vault = address(new Vault(address(token0), address(token1)));
            strategy = address(new Strategy(address(pool), address(vault), rebalancer, feeRecipient));
            IVault(vault).setStrategy(strategy);
            vm.stopPrank();
        }

        // initial deposit
        {
            vm.startPrank(owner);
            uint256 amount0 = 1000 * (10 ** token0.decimals());
            uint256 amount1 = 1000 * (10 ** token1.decimals());

            deal(address(token0), owner, amount0);
            deal(address(token1), owner, amount1);
            IERC20(token0).safeIncreaseAllowance(address(vault), type(uint256).max);
            IERC20(token1).safeIncreaseAllowance(address(vault), type(uint256).max);
            IVault(vault).deposit(amount0, amount1, 0, 0);
            vm.stopPrank();
        }

        // initial rebalance
        {
            vm.startPrank(rebalancer);
            skip(10 minutes);
            IStrategy(strategy).rebalance();
            vm.stopPrank();
        }

        // vesting position
        {
            vm.startPrank(owner);

            deal(address(token0), owner, 1 ether);
            deal(address(token1), owner, 1 ether);
            IERC20(token0).safeIncreaseAllowance(address(vault), type(uint256).max);
            IERC20(token1).safeIncreaseAllowance(address(vault), type(uint256).max);
            IVault(vault).addVestingPosition(1 ether, 1 ether, 365 days);

            vm.stopPrank();
        }

        // move price
        {
            address swapper = vm.createWallet("swapper").addr;
            vm.startPrank(swapper);
            uint256 swapAmount = 100 ether;
            deal(address(token1), swapper, swapAmount);
            IERC20(token1).safeIncreaseAllowance(address(UNISWAP_V3_SWAP_ROUTER), type(uint256).max);
            ISwapRouter(UNISWAP_V3_SWAP_ROUTER).exactInputSingle(
                ISwapRouter.ExactInputSingleParams({
                    tokenIn: pool.token1(),
                    tokenOut: pool.token0(),
                    fee: pool.fee(),
                    recipient: swapper,
                    amountIn: swapAmount,
                    amountOutMinimum: 0,
                    sqrtPriceLimitX96: 0,
                    deadline: type(uint256).max
                })
            );

            vm.stopPrank();
        }

        // second rebalance
        {
            vm.startPrank(rebalancer);
            skip(10 minutes);
            IStrategy(strategy).rebalance();
            vm.stopPrank();
        }


        // fees
        {
            address swapper = vm.createWallet("swapper").addr;
            vm.startPrank(swapper);
            uint256 swapAmount = 270000e6;
            deal(address(token0), swapper, swapAmount);
            IERC20(token0).safeIncreaseAllowance(address(UNISWAP_V3_SWAP_ROUTER), type(uint256).max);
            ISwapRouter(UNISWAP_V3_SWAP_ROUTER).exactInputSingle(
                ISwapRouter.ExactInputSingleParams({
                    tokenIn: pool.token0(),
                    tokenOut: pool.token1(),
                    fee: pool.fee(),
                    recipient: swapper,
                    amountIn: swapAmount,
                    amountOutMinimum: 0,
                    sqrtPriceLimitX96: 0,
                    deadline: type(uint256).max
                })
            );

            vm.stopPrank();
        }

        vm.expectRevert(bytes("NP"), address(pool));
        IStrategy(strategy).collectFees();
    }
}