# 🔍 Finding Title

## 📌 Summary  
_What was the issue?_  

    When calculating the borrowed value we don't devide by the denominator 10^decimals(), but we devide only by decimals() which makes the number insanly high.
    uint256 borrowedValue = owedAmount * bPrice / ERC20(up.denomination).decimals();

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

    I overloocked the calculations maybe need a better way to keep knowladge of the denominators of values.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

    Test the calculations more in depth and don't overlook simple looking calculations.
    
## 🔎 The Finding

Liquidation fee will not be claimed due to incorrect decimal handling
Summary
Incorrect decimal handling in liquidation logic will prevent liquidation fee recipient from receiving liquidation fees.

Root Cause
In Yieldoor protocol, liquidation fee is claimed in the following case:

Position's current USD value is greater than borrowed token's current USD value

However, borrowed token's current usd Value is inflated in Leverager.sol:318

uint256 borrowedValue = owedAmount * bPrice / ERC20(up.denomination).decimals();
Instead of dividing by 10 ^ decimals, it divides by decimals.

Thus, borrowedValue is greatly inflated.

And the following if branch will never get reached:

@>      if (totalValueUSD > borrowedValue) {
            // What % of the amountsOut are profit is calculated by `(totalValueUSD - borrowedUSD) / totalValueUSD`
            // Then, on top of that, we calculate the protocol fee and scale it in 1e18.

            uint256 protocolFeePct = 1e18 * liquidationFee * (totalValueUSD - borrowedValue) / (totalValueUSD * 10_000);
            uint256 pf0 = protocolFeePct * amount0 / 1e18;
            uint256 pf1 = protocolFeePct * amount1 / 1e18;

            if (pf0 > 0) IERC20(up.token0).safeTransfer(feeRecipient, pf0);
            if (pf1 > 0) IERC20(up.token1).safeTransfer(feeRecipient, pf1);
            amount0 -= pf0;
            amount1 -= pf1;
        }
Internal Pre-conditions
Liquidator liquidates a position
Position's total value in USD is greater than borrowed value in USD
External Pre-conditions
N/A

Attack Path
N/A

Impact
Liquidation fee is not collected.

PoC
Scenario

Liquidation fee is set to 900 BPS
A leverager (depositor) creates the following leveraged position:
token0 = WBTC
token1 = USDC
amount0In = 0
amoutn1In = 100000 USDC
denomination = WETH
vault0In = 1 WBTC = 100000 USD
vault1In = 100000 USDC = 100000 USD
maxBorrowAmount = 41 WETH
Total position value is 200k USD
Total borrowed amount is 88800 USD (or 35.52 WETH)
Here, debt is lower than WBTC price because Uniswap V3 is using current WBTC price (88800 USD) instead of test price (100000 USD)
Later WBTC price drops to 40000 USD and USDC price drops to 0.5 USD
Position total value is 90000 USD and liquidatable
Liquidator liquidates the position
Liquidation fee recipient receives nothing although position value is greater than borrowed value
How to run POC

In order to check how much borrowedValue is inflated, you can optionally apply the following patch:

diff --git a/yieldoor/src/Leverager.sol b/yieldoor/src/Leverager.sol
index b59c56c..bcad781 100644
--- a/yieldoor/src/Leverager.sol
+++ b/yieldoor/src/Leverager.sol
@@ -292,6 +292,7 @@ contract Leverager is ReentrancyGuard, Ownable, ERC721, ILeverager {
         }
     }
 
+    event log_named_decimal_uint(string key, uint256 value, uint256 decimal);
     /// @notice Liquidates a certain leveraged position.
     /// @dev Check the ILeverager contract for comments on all LiquidateParams arguments
     /// @dev Does not support partial liquidations
@@ -317,6 +318,8 @@ contract Leverager is ReentrancyGuard, Ownable, ERC721, ILeverager {
         uint256 bPrice = IPriceFeed(pricefeed).getPrice(up.denomination);
         uint256 borrowedValue = owedAmount * bPrice / ERC20(up.denomination).decimals();
 
+        emit log_named_decimal_uint("totalValueUSD", totalValueUSD, 18);
+        emit log_named_decimal_uint("borrowedValue", borrowedValue, 18);
         if (totalValueUSD > borrowedValue) {
             // What % of the amountsOut are profit is calculated by `(totalValueUSD - borrowedUSD) / totalValueUSD`
             // Then, on top of that, we calculate the protocol fee and scale it in 1e18.
Add the following content to Leverager.t.sol:

    function test_cantClaimLiquidationFee() external {
        vm.startPrank(owner);
        ILeverager(leverager).enableTokenAsBorrowed(address(weth));
        ILeverager(leverager).setLiquidationFee(900);
        vm.stopPrank();
        // depositor initial fund 100000 USDC = 100000 USD
        vm.startPrank(depositor);
        deal(address(wbtc), depositor, 0e8);
        deal(address(usdc), depositor, 100_000e6);

        // position will have 1WBTC and 100000 USDC
        // total position value would be 200000 USD
        // denom token is weth, will borrow some weth and swap them into wbtc
        ILeverager.LeverageParams memory lp;
        lp.amount0In = 0e8;
        lp.amount1In = 100_000e6;
        lp.vault0In = 1e8;
        lp.vault1In = 100_000e6;
        lp.vault = vault;
        lp.maxBorrowAmount = 41e18;
        lp.denomination = address(weth);

        IMainnetRouter.ExactOutputParams memory ep1;
        ep1.path = abi.encodePacked(address(wbtc), uint24(3000), address(weth));
        ep1.deadline = block.timestamp + 300;
        ep1.amountInMaximum = 41e18;
        ep1.recipient = leverager;

        lp.swapParams1 = abi.encode(ep1);

        wbtc.approve(leverager, type(uint256).max);
        usdc.approve(leverager, type(uint256).max);

        ILeverager(leverager).openLeveragedPosition(lp);
        vm.stopPrank();

        ILeverager.Position memory position = ILeverager(leverager).getPosition(1);
        // debt is 88800 USD
        assertEq(position.initBorrowedUsd / 10 ** 18, 88800);

        // price changes will drop position value to 90000 USD
        // position is liquidatable but still position value is greater than debt
        // so liquidation fee is expected to be claimed
        MockOracle(wbtcOracle).setPrice(40000e18); // position has 1 WBTC = 40000 USD
        MockOracle(usdcOracle).setPrice(0.5e18); // position has 100000 USDC = 50000 USD

        // liquidator liquidate the position with denom token - weth
        vm.startPrank(liquidator);
        uint256 debtWethAmount = position.borrowedAmount
            * ILendingPool(lendingPool).getCurrentBorrowingIndex(position.denomination) / position.borrowedIndex;
        deal(address(weth), liquidator, debtWethAmount);
        weth.approve(address(leverager), debtWethAmount);
        ILeverager.LiquidateParams memory lip;
        lip.id = 1;
        ILeverager(leverager).liquidatePosition(lip);
        vm.stopPrank();

        // fee recipient received nothing
        assertEq(wbtc.balanceOf(ILeverager(leverager).feeRecipient()), 0);
        assertEq(usdc.balanceOf(ILeverager(leverager).feeRecipient()), 0);
    }
Run the following command:

forge test --rpc-url $MAINNET_FORK_URL  --match-test test_cantClaimLiquidationFee --fork-block-number 21926708 -vvv
Console Output:

[PASS] test_cantClaimLiquidationFee() (gas: 1862589)
Logs:
  totalValueUSD: 89999.999199500000000000
  borrowedValue: 4933386117326049569027.777777777777777777
Mitigation
The following can fix the calculation

- uint256 borrowedValue = owedAmount * bPrice / ERC20(up.denomination).decimals();
+ uint256 borrowedValue = owedAmount * bPrice / 10 ** ERC20(up.denomination).decimals();