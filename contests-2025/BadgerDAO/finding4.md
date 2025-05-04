# 🔍 Finding Title Migration will fail indefinitely if EXTERNAL_VAULT.redeem reverts and funds will be indefinitely stuck in the vault

## 📌 Summary  
_What was the issue?_  

On migration it is assumed that the redeem function can't revert which is not the case and the migration will be stuck which may cause lock of funds.

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

Didn't take into account that this assumes that this function won't revert.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

Don't assume anything

## 🔎 The Finding

Summary
In the Notes.md, it is stated that one of the invariants that should never fail is migration. However, the escrow calls __beforeMigration which calls EXTERNAL_VAULT.redeem() and it trusts that it never reverts. This function is likely to fail since the vaults mentioned in the readme as "ERC4626 of interest" can revert redeems due to various reasons ( further explanation later ) , this leads to delay migration operations ( sometimes undefinetely ), which sometimes can be urgent or critical, and lock the funds in the external vault for an undefined amount of time.

Finding Description
The function updateEscrow() calls onMigrateSource() in EbtcBSM.sol:383. onMigrateSource() calls _beforeMigration() which performs a redeem on the external vault. The code trusts that redeem() can never revert, I checked the vaults mentioned as "ERC4626 of interest" and found that redeems are likely to revert due to various reasons:

1 - The vaults can be paused or frozen
Let's take Aave-Vault for example, you'll find checks whether the vault is paused, frozen, or inactive before any operation, by using variables like AAVE_ACTIVE_MASK, AAVE_FROZEN_MASK, AAVE_PAUSED_MASK, for example in the line 610 in the function _maxAssetsWithdrawableFromAave() which is used in redeems. Here's a snippet from _handleRedeem which performs redeems :

    function _handleRedeem(
        uint256 shares,
        address receiver,
        address owner,
        address allowanceTarget,
        bool asAToken
    ) internal returns (uint256) {
        _accrueYield();
        require(shares <= maxRedeem(owner), "REDEEM_EXCEEDS_MAX");
        uint256 assets = super.previewRedeem(shares);
        require(assets != 0, "ZERO_ASSETS"); // Check for rounding error since we round down in previewRedeem.
        _baseWithdraw(assets, shares, owner, receiver, allowanceTarget, asAToken);
        return assets;
    }

If the vault is paused or frozen , maxRedeem() will return 0 , ( check the comments of _maxAssetsWithdrawableFromAave() which is used in maxRedeem ). This will cause the migration to fail indefinitely and funds to be locked indefinitely.

2 - Our shares > maxRedeem
Since _beforeMigration uses EXTERNAL_VAULT.balanceOf(address(this)) as shares, this value may be greater than maxRedeem ( maxRedeem doesn't return our shares , it returns the minimum of available liquidity and our shares, check this comment ) . If liquidity drops below the amount needed to redeem all shares owned by ERC4626Escrow, the migration will always fail until there is enough liquidity again in the vault.

You can see here that we are relying on external circumstances and external users' behaviors to determine the health of our system.

3 - assets == 0
This is very likely to happen, if we have not deposited in the vault (or deposited and redeemed), the condition require(assets != 0, "ZERO_ASSETS") will revert causing migration to temporarily fail. However this is not as impactful since we can manually deposit again and then call updateEscrow(). However this could lead to a slight delay or a slight loss of assets.

Impact Explanation
1 - The vaults can be paused or frozen
In this case we don't know for how long the migration will keep failing, the migration of our own system relies on the actions taken by people outside the protocol.
This will reduce liquidity of our protocol, causing buyAsset to fail sometimes since _ensureLiquidity() also calls redeem.
So the impact here will be High.

2 - Our shares > maxRedeem
Here also we don't know for how long the migration will keep failing, the migration of our own system relies on users' behavior on another vault which belongs to another protocol.
Consider a scenario where you notice that liquidity dropped in a certian vault, you want to migrate to another but you won't be able to since shares > maxRedeem ( because of the use of EXTERNAL_VAULT.balanceOf(address(this)) as shares ). ( there is no other way to update the vault, it's immutable ).
Also reduces liquidity causing buyAsset to fail sometimes since _ensureLiquidity() also calls redeem.
So the impact here will be High.

3 - assets == 0
Authorized users can deposit and then migrate, it leads to a slight delay in migration and maybe a slight loss of assets, so Low impact here.

Since funds can be locked indefinitely in two scenarios and migration can be blocked for an unknown amount of time, and sometimes migration can be critical or urgent the final impact is High.

Likelihood Explanation
There are several scenarios in which redeem fails and not only one. And all scenarios are not away from happening, but they are not all an occurrence that happens all the time, so i'm setting the likelihood as Medium.

Proof of Concept
I simulated the redeem function of the vaults of interest using a mock since the mock used by the devs does not correspond to the behaviour of the vaults of interest.

1 - Please place the following ERC4626Mock.sol file under your test/mocks directory :

// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import {ERC4626Fees} from "@openzeppelin/contracts/mocks/docs/ERC4626Fees.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";

contract MockERC4626Fees is ERC4626Fees, Pausable {
    uint256 private immutable _entryFeeBasisPointValue;
    address private immutable _entryFeeRecipientValue;
    uint256 private immutable _exitFeeBasisPointValue;
    address private immutable _exitFeeRecipientValue;

    constructor(
        IERC20 asset,
        uint256 entryFeeBasisPoints,
        address entryFeeRecipient,
        uint256 exitFeeBasisPoints,
        address exitFeeRecipient
    ) ERC4626(asset) ERC20("MockERC4626Fees", "MEF") { 
        _entryFeeBasisPointValue = entryFeeBasisPoints;
        _entryFeeRecipientValue = entryFeeRecipient;
        _exitFeeBasisPointValue = exitFeeBasisPoints;
        _exitFeeRecipientValue = exitFeeRecipient;
    }

    function _entryFeeBasisPoints() internal view virtual override returns (uint256) {
        return _entryFeeBasisPointValue;
    }

    function _entryFeeRecipient() internal view virtual override returns (address) {
        return _entryFeeRecipientValue;
    }

    function _exitFeeBasisPoints() internal view virtual override returns (uint256) {
        return _exitFeeBasisPointValue;
    }

    function _exitFeeRecipient() internal view virtual override returns (address) {
        return _exitFeeRecipientValue;
    }

    function redeem(uint256 shares, address receiver, address owner) public override whenNotPaused returns (uint256) {
        require(convertToAssets(shares) != 0,"Cannot redeem 0");
        
        super.redeem(shares, receiver, owner);
    }

    function pause() public {
        _pause();
    }

    function unpause() public {
        _unpause();
    }
}

2 - Put the following Audit.t.sol under your test/ directory :

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import "./BSMTestBase.sol";
import {IEbtcBSM} from "../src/Dependencies/IEbtcBSM.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/mocks/token/ERC4646FeesMock.sol";
import "@openzeppelin/contracts/mocks/token/ERC4626Mock.sol";
import {ERC4626Escrow} from "../src/ERC4626Escrow.sol";
import {MockERC4626Fees} from "./mocks/ERC4626Mock.sol";
import {ERC20Mock} from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import {ERC4626Mock} from "@openzeppelin/contracts/mocks/token/ERC4626Mock.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";

contract AuditTest is BSMTestBase {
    event AssetBought(uint256 ebtcAmountIn, uint256 assetAmountOut, uint256 feeAmount);
    event FeeToBuyUpdated(uint256 oldFee, uint256 newFee);
    address user1 = makeAddr("user1");
    ERC4626Escrow newEscrow;
    MockERC4626Fees vault;

    // The vault ERC4626Mock used in the tests by the devs does not match the behavior of the actual vaults
    // of interest mentioned in the readme. So we will use our own mock to simulate it.
    // This modifier is to prepare a vault that actually matches the behavior of the vaults of interest
    modifier prepareNewVault() {
        vault = new MockERC4626Fees(
            IERC20(mockAssetToken),0, address(0x1337), 0, address(mockAssetToken) 
        );
        escrow = new ERC4626Escrow(
            address(vault),
            address(mockAssetToken),
            address(bsmTester),
            address(authority),
            address(defaultFeeRecipient)
        );
        setRoleCapability(
            15,
            address(escrow),
            newEscrow.depositToExternalVault.selector,
            true
        );
        vm.startPrank(address(techOpsMultisig));
        bsmTester.updateEscrow(address(escrow));
        vm.stopPrank();
        _;
    }

    function testMigrationFailsDueToPause() public prepareNewVault {
        vm.startPrank(user1);
        deal(address(escrow.ASSET_TOKEN()), user1, 10 ether);
        IERC20(address(escrow.ASSET_TOKEN())).approve(address(bsmTester), type(uint256).max);
        bsmTester.sellAsset(7 ether, user1, 0);
        vm.stopPrank();


        newEscrow = new ERC4626Escrow(
            address(vault),
            address(mockAssetToken),
            address(bsmTester),
            address(authority),
            address(defaultFeeRecipient)
        );
        vm.startPrank(address(techOpsMultisig));
        vault.pause();
        escrow.depositToExternalVault(1 ether, 0);
        vm.expectRevert(Pausable.EnforcedPause.selector);
        bsmTester.updateEscrow(address(newEscrow));
        vm.stopPrank();
    }
        

    function testMigrationFailsDueTo0Shares() public prepareNewVault {
        vm.startPrank(user1);
        deal(address(escrow.ASSET_TOKEN()), user1, 10 ether);
        IERC20(address(escrow.ASSET_TOKEN())).approve(address(bsmTester), type(uint256).max);
        bsmTester.sellAsset(7 ether, user1, 0);
        vm.stopPrank();


        newEscrow = new ERC4626Escrow(
            address(vault),
            address(mockAssetToken),
            address(bsmTester),
            address(authority),
            address(defaultFeeRecipient)
        );
  
        vm.startPrank(address(techOpsMultisig));
        vm.expectRevert("Cannot redeem 0");
        bsmTester.updateEscrow(address(newEscrow));
        vm.stopPrank();
    }
}

3 - Run forge test --mt testMigrationFailsDueToPause -vvvv , and notice that it passes and the migration reverts.

4 - Run forge test --mt testMigrationFailsDueTo0Shares -vvvv , and notice that it passes and the migration reverts.

Recommendation
Don't make migration reliant on redeem. Instead, you can transfer the shares to a trusted address in _beforeMigration() ( since shares are ERC20 ), and then redeem them later and deposit redeemed assets in the new escrow.