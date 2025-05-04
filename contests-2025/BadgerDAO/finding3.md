# 🔍 Finding Title Improper staleness check for asset price

## 📌 Summary  
_What was the issue?_  

Wrong check for stale data because updated_at is set to the min of tBTC/usd and BTC/usd which have different heartbeats which makes this inconsitent because one can be stale.

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

I thought this is how the protocol should be. Never assume anything.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

Don't fear writing reports and report more things.

## 🔎 The Finding

Description
When a user sells assets to EbtcBSM, protocol will check oracle price constraint to determine if minting is allowed based on the current asset price.

OraclePriceConstraint::canMint():

    function canMint(
        uint256 _amount,
        address _minter
    ) external view returns (bool, bytes memory) {
@>      uint256 assetPrice = _getAssetPrice();
        /// @dev peg price is 1e18
        uint256 minAcceptablePrice = (1e18 * minPriceBPS) / BPS;

        if (minAcceptablePrice <= assetPrice) {
            return (true, "");
        } else {
            return (
                false,
                abi.encodeWithSelector(
                    BelowMinPrice.selector,
                    assetPrice,
                    minAcceptablePrice
                )
            );
        }
    }

The asset token is expected to be tBTC or cbBTC, and the asset price is retrieved from ASSET_FEED which returns tBTC/BTC or cbBTC/BTC price.

OraclePriceConstraint::_getAssetPrice():

    function _getAssetPrice() private view returns (uint256) {
@>      (, int256 answer, , uint256 updatedAt, ) = ASSET_FEED.latestRoundData();

        if (answer <= 0) revert BadOraclePrice(answer);
@>      if ((block.timestamp - updatedAt) > oracleFreshnessSeconds) {
@>          revert StaleOraclePrice(updatedAt);
@>      }

        return (uint256(answer) * 1e18) / ASSET_FEED_PRECISION;
    }

Under the hood, ASSET_FEED retrieves the latest price of tBTC/USD or cbBTC/USD from Chainlink Price Feed, then convert it with BTC/USD price to get tBTC/BTC or cbBTC/BTC price, and returns the price and updateAt to OraclePriceConstraint.

tBTCChainlinkAdapter::latestRoundData():

    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        )
    {
        (int256 tBtcUsdPrice, uint256 tBtcUsdUpdatedAt) = _latestRoundData(TBTC_USD_CL_FEED);
        (int256 btcUsdPrice, uint256 btcUsdUpdatedAt) = _latestRoundData(BTC_USD_CL_FEED);

@>      updatedAt = _min(tBtcUsdUpdatedAt, btcUsdUpdatedAt);
@>      answer = _convertAnswer(btcUsdPrice, tBtcUsdPrice);
    }

As can be seen, updatedAt is determined by the minimum value of tBtcUsdUpdatedAt and btcUsdUpdatedAt, and in OraclePriceConstraint, the value is compared with oracleFreshnessSeconds.

        if (answer <= 0) revert BadOraclePrice(answer);
        if ((block.timestamp - updatedAt) > oracleFreshnessSeconds) {
            revert StaleOraclePrice(updatedAt);
        }

In Chainlink, BTC/USD's deviation is 0.5% and heartbeat is 3600s:

BTC:USD.png

Both tBTC/USD and cbBTC/USD's deviation is 2% and heartbeat is 86400s: tBTC:USD.png

cbBTC:USD.png

Because the price is converted from BTC/USD and tBTC/USD( or cbBTC/USD), and oracleFreshnessSeconds is used to check the age of the latest oracle price, this may lead to stale BTC price be used or the transaction may always revert.

Consider the following scenarios:

oracleFreshnessSeconds is set to 86400s, if BTC/USD is updated at 3600s ago, this means BTC/USD price is stale, however, the transaction will not revert if tBTC/USD( or cbBTC/USD) is updated at within 86400s till now;

oracleFreshnessSeconds is set to 3600s, if tBTC/USD( or cbBTC/USD) is updated at 3600s ago, the transaction will revert as the protocol assumes the price is stale, however, this check is too strict given tBTC/USD( orcbBTC/USD)'s deviation is 2% and heartbeat is 86400s.

Proof of Concept
Please run forge test --mt testAudit_StalenessCheck.

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import {Test, console} from "forge-std/Test.sol";

import "../src/BSMDeployer.sol";
import "../src/EbtcBSM.sol";
import "../src/ERC4626Escrow.sol";
import "../src/OraclePriceConstraint.sol";
import "../src/RateLimitingConstraint.sol";
import "../src/tBTCChainlinkAdapter.sol";
import "../src/ActivePoolObserver.sol";
import "../src/Dependencies/Authority.sol";

contract Audit is Test {
    struct MarketParams {
        address loanToken;
        address collateralToken;
        address oracle;
        address irm;
        uint256 lltv;
    }

    Authority governance;
    address feeRecipient = makeAddr("FeeRecipient");

    address tBTC = 0x18084fbA666a33d37592fA2633fD49a74DD93a88;
    address cbBTC = 0xcbB7C0000aB88B473b1f5aFd9ef808440eed33Bf;
    address eBTC = 0x661c70333AA1850CcDBAe82776Bb436A0fCfeEfB;

    EbtcBSM eBtcBsm;
    ERC4626Escrow escrow;

    BSMDeployer bsmDeployer;
    OraclePriceConstraint oraclePriceConstraint;
    RateLimitingConstraint rateLimitingConstraint;
    tBTCChainlinkAdapter tBTCAdapter;

    AggregatorV3Interface tBtcUsdClFeed;
    AggregatorV3Interface btcUsdClFeed;

    ActivePoolObserver activePoolObserver;

    function setUp() public {
        vm.createSelectFork("https://eth.blockrazor.xyz");

        // Governance
        (bool success, bytes memory result) = eBTC.call(abi.encodeWithSignature("authority()"));
        require(success, "Get eBTC authority failed");
        governance = Authority(abi.decode(result, (address)));
        vm.mockCall(
            address(governance),
            abi.encodeWithSelector(Authority.canCall.selector),
            abi.encode(true)
        );

        tBtcUsdClFeed = AggregatorV3Interface(0x8350b7De6a6a2C1368E7D4Bd968190e13E354297);
        btcUsdClFeed = AggregatorV3Interface(0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c);

        // Oracle
        tBTCAdapter = new tBTCChainlinkAdapter(tBtcUsdClFeed, btcUsdClFeed);
        oraclePriceConstraint = new OraclePriceConstraint(address(tBTCAdapter), address(governance));

        address observer = 0x6dBDB6D420c110290431E863A1A978AE53F69ebC; // ActivePool
        activePoolObserver = new ActivePoolObserver(ITwapWeightedObserver(observer));
        rateLimitingConstraint = new RateLimitingConstraint(address(activePoolObserver), address(governance));

        // Vault
        address _externalVault = makeAddr("Vault");

        // BSM
        eBtcBsm = new EbtcBSM(
            tBTC,
            address(oraclePriceConstraint),
            address(rateLimitingConstraint),
            eBTC,
            address(governance)
        );

        escrow = new ERC4626Escrow(
            _externalVault,
            tBTC,
            address(eBtcBsm),
            address(governance),
            feeRecipient
        );

        eBtcBsm.initialize(address(escrow));

        // Set minting config
        RateLimitingConstraint.MintingConfig memory mintingConfig = RateLimitingConstraint.MintingConfig({
            relativeCapBPS: 1000,
            absoluteCap: 100e18,
            useAbsoluteCap: true
        });
        vm.startPrank(address(governance));
        rateLimitingConstraint.setMintingConfig(address(eBtcBsm), mintingConfig);
        vm.stopPrank();

        vm.label(address(governance), "Governance");
        vm.label(feeRecipient, "FeeRecipient");
        vm.label(tBTC, "tBTC");
        vm.label(eBTC, "eBTC");
        vm.label(cbBTC, "cbBTC");
        vm.label(address(eBtcBsm), "EbtcBSM");
        vm.label(address(escrow), "ERC4626Escrow");
        vm.label(address(bsmDeployer), "BSMDeployer");
        vm.label(address(oraclePriceConstraint), "OraclePriceConstraint");
        vm.label(address(rateLimitingConstraint), "RateLimitingConstraint");
        vm.label(address(tBTCAdapter), "tBTCChainlinkAdapter");
        vm.label(address(activePoolObserver), "ActivePoolObserver");
    }

    function testAudit_StalenessCheck() public {
        uint256 sellAmount = 1e18;
        address alice = makeAddr("Alice");
        deal(tBTC, alice, sellAmount * 2);

        // `oracleFreshnessSeconds` is default to 1 day
        uint256 oracleFreshnessSeconds = oraclePriceConstraint.oracleFreshnessSeconds();
        assertEq(oracleFreshnessSeconds, 86400);

        // tBTC/USD
        vm.mockCall(
            address(tBtcUsdClFeed),
            abi.encodeWithSelector(AggregatorV3Interface.latestRoundData.selector),
            abi.encode(1, 100000e8, block.timestamp, block.timestamp, 1)
        );
        // BTC/USD is updated at 2 hours ago and the price is stale
        vm.mockCall(
            address(btcUsdClFeed),
            abi.encodeWithSelector(AggregatorV3Interface.latestRoundData.selector),
            abi.encode(1, 100000e8, block.timestamp - 2 hours, block.timestamp - 2 hours, 1)
        );

        // Sell Asset
        vm.startPrank(alice);
        IERC20(tBTC).approve(address(eBtcBsm), sellAmount);
        // Transaction is not revert and stale BTC price is used
        eBtcBsm.sellAsset(sellAmount, alice, 0);
        vm.stopPrank();

        // Set `oracleFreshnessSeconds` to 1 hour
        oracleFreshnessSeconds = 3600;
        oraclePriceConstraint.setOracleFreshness(oracleFreshnessSeconds);

        // tBTC/USD is udpated at 2 hours ago but the price is not stale
        vm.mockCall(
            address(tBtcUsdClFeed),
            abi.encodeWithSelector(AggregatorV3Interface.latestRoundData.selector),
            abi.encode(1, 100000e8, block.timestamp - 2 hours, block.timestamp - 2 hours, 1)
        );
        // BTC/USD
        vm.mockCall(
            address(btcUsdClFeed),
            abi.encodeWithSelector(AggregatorV3Interface.latestRoundData.selector),
            abi.encode(1, 100000e8, block.timestamp, block.timestamp, 1)
        );

        // Sell Asset
        vm.startPrank(alice);
        IERC20(tBTC).approve(address(eBtcBsm), sellAmount);
        // Transaction is revert due to staleness check but the price is not actually stale
        vm.expectRevert(abi.encodeWithSignature("StaleOraclePrice(uint256)", block.timestamp - 2 hours));
        eBtcBsm.sellAsset(sellAmount, alice, 0);
        vm.stopPrank();
    }
}

Recommendation
It is recommended to use separate oracle price checks for the staleness check of BTC/USD and tBTC/USD( orcbBTC/USD).