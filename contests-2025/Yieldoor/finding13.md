# 🔍 Finding Title

## 📌 Summary  
_What was the issue?_  

    Incorect path set when multiple swaps are done not updated and use the same array over and over

         while (path.hasMultiplePools()) {
+             path = path.skipToken();
-             path.skipToken();
        }

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

    Didn't think through the scenario for multiple swaps

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

    Go deeper in funtions that seem easy.

## 🔎 The Finding

Multi-hop swaps are not properly supported, the code will run OOG
Summary
Multi-hop swaps are not properly supported, the code will run OOG

Root Cause
When calling Leverager::_getTokenIn(), we have the following code:

while (path.hasMultiplePools()) {
            path.skipToken();
}
The issue is that we use the same path for each iteration, thus we are simply repeating the same iteration over and over again, resulting in OOG.

Internal Pre-conditions
No internal pre-conditions

External Pre-conditions
No external pre-conditions

Attack Path
User wants to open a leverage position and has to conduct a multi-hop swap for the leverage (supported by the code)
Code will run OOG
Impact
Loss of funds due to the OOG, as well as broken functionality rendering the functionality useless.

PoC
Paste the following test into Remix:

// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8 <0.9.0;

import "hardhat/console.sol";
import "./Path.sol";

contract Testing {
    using Path for bytes;
    function tryOOG() public pure {
        bytes memory path = hex"C02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2000bb8A0b86991c6218b36c1d19D4a2e9Eb0cE3606eB480001f46B175474E89094C44Da98b954EedeAC495271d0F";

        while (path.hasMultiplePools()) {
            path.skipToken();
        }
    }
}
Mitigation
        while (path.hasMultiplePools()) {
+             path = path.skipToken();
-             path.skipToken();
        }