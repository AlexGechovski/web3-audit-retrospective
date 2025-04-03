# 🔍 Finding Title

## 📌 Summary  
_What was the issue?_  

    feeRecipient is never initialized and theres no setter.

## ❌ Why I Missed It  
_What caused me to overlook this issue?_ 

    Didn't check where it is set.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

    Check variables for missed setters.

## 🔎 The Finding

Uninitialized feeRecipient Prevents Liquidations and Risks Protocol Insolvency
Summary
The feeRecipient variable exists but is never initialized.
When liquidations occur, the contract tries to send tokens to this address.
Transfers fail, preventing liquidators from recovering bad debt.
This leads to protocol insolvency due to stuck liquidations.

Root Cause
The root cause of the issue is the failure to initialize the feeRecipient variable, which results in an invalid recipient address during liquidation transfers.

In Leverager.sol, the feeRecipient variable exists but is never initialized in the constructor or via a setter function.
This means the contract defaults to transferring funds to address(0) or an uninitialized storage value.
When a liquidation occurs, the contract attempts to send tokens to feeRecipient, causing a failed transfer and a reverted transaction.

https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/Leverager.sol#L320

#L320:

if (totalValueUSD > borrowedValue) {
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
feeRecipient is Never Initialized
Liquidation Function Attempts to Transfer Fees
No Fallback Mechanism Exists
At Least One Position Must Be Liquidatable
Liquidators Must Attempt a Liquidation
External Pre-conditions
A Liquidation Must Be Triggered
A Liquidator Must Attempt to Liquidate a Position
The Network Must Process the Liquidation Transaction
There Must Be Open Positions with Debt
Attack Path
A Borrower’s Position Becomes Undercollateralized
A Liquidator Calls liquidate()
The Contract Attempts to Transfer Fees to feeRecipient
The Transaction Reverts
Repeatable Failure Leading to Protocol Insolvency
Impact
The protocol cannot process liquidations, leading to the accumulation of bad debt.
Liquidators cannot execute successful liquidations, causing them to miss out on rewards.
The protocol suffers potential insolvency as undercollateralized positions remain open indefinitely.
PoC
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {Leverager} from "../src/Leverager.sol";

contract StandaloneFeeRecipientTest is Test {
    Leverager public leverager;

    function setUp() public {
        // Deploy a minimal Leverager instance
        leverager = new Leverager("Leveraged Position", "LP", address(0x123));
    }

    function test_LeveragerStorageScan() public {
        // Set liquidation fee to a distinct value
        vm.startPrank(leverager.owner());
        leverager.setLiquidationFee(1234); // Distinct value to find
        vm.stopPrank();
        
        // Before setting, print address slots to find fee recipient
        console.log("Scanning for address slots (possible feeRecipient locations):");
        for (uint256 i = 0; i < 20; i++) {
            bytes32 value = vm.load(address(leverager), bytes32(i));
            address addr = address(uint160(uint256(value)));
            // Only print if it looks like an address (has some bits set in address range)
            if (uint256(value) > 0 && uint256(value) < 2**160) {
                console.log("Slot", i, ":", addr);
            }
        }
        
        // Find liquidation fee location
        console.log("\nScanning for liquidationFee (value 1234):");
        for (uint256 i = 0; i < 20; i++) {
            bytes32 value = vm.load(address(leverager), bytes32(i));
            if (uint256(value) == 1234) {
                console.log("Found liquidationFee at slot:", i);
            }
        }
        
        assertTrue(true, "Test completed storage scan");
    }
}
Image

Mitigation
Constructor parameter for the fee recipient:

constructor(string memory name_, string memory symbol_, address _lendingPool, address _feeRecipient)
    Ownable(msg.sender)
    ERC721(name_, symbol_)
{
    lendingPool = _lendingPool;
    feeRecipient = _feeRecipient;
    require(_feeRecipient != address(0), "Fee recipient cannot be zero address");
}`

Setter function with proper access control:

`function setFeeRecipient(address _feeRecipient) external onlyOwner {
    require(_feeRecipient != address(0), "Fee recipient cannot be zero address");
    feeRecipient = _feeRecipient;
}