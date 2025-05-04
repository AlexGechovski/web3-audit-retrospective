# 🔍 Finding Title Morpho rewards stuck into the escrow contract

## 📌 Summary  
_What was the issue?_  


The protocol states that it should integrade with Morpho vaults, but morpho can give rewards in a different erc20 token then the protocol expectes and it's not accounted for. This can cause loss of funds

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

Didn't know morpho can distrubute reward tokens in a different token then the vault token.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

Research better integrations with other protocols that should be integrated to work with.

## 🔎 The Finding


Summary
If an owner of a Metamorpho vault decides to redistribute the rewards received from Morpho Blue, and these rewards are different from the asset managed by the escrow contract, they will become stuck in the contract.

Finding Description
Based on the README of the contest:

Integration Targets
The ERC4626 of interest are:

AAVE (via this adapter: https://github.com/aave/Aave-Vault)
Morpho
Euler
All these 3 conform to the OpenZeppelin ERC4626 implementation

Due to a non-trivial amount of integration risks, we expect to exclusively integrate with vaults that conform to the OZ ERC4626 implementation

We can consider Metamorpho Vaults to be in scope, as they are fully compliant with OZ ERC4626 and are developed by the Morpho protocol itself.

Now, referring to the README of Metamorpho:

The vault may be entitled to some rewards emitted on Morpho Blue markets the vault has supplied to. Those rewards can be transferred to the skimRecipient. The vault's owner has the choice to distribute back these rewards to vault depositors however they want. For more information about this use case, see the Rewards section.

Rewards
To redistribute rewards to vault depositors, it is advised to use the Universal Rewards Distributor (URD).

The URD is also developed by the Morpho protocol and is designed to distribute multiple ERC20 tokens from a single off-chain computed Merkle tree.

Inside of UniversalRewardsDistributor.sol::claim(), anyone can claim rewards on behalf of an account. However, since the escrow contract lacks functionality to claim the rewards automatically, someone must manually execute the claim on its behalf to transfer the rewards to the escrow contract.

The problem arises when the claimed rewards are in a different token than the asset managed by the escrow contract. Since the escrow also lacks hooks to handle other ERC20 tokens, these rewards will remain stuck in the contract, effectively causing the loss of funds.

Impact Explanation
High -- Loss of funds for the protocol

Likelihood Explanation
Low -- Owner of the Metamorpho needs to decide to redistribute the rewards to vault depositors. But if the case, all the rewards will be lost.

Recommendation
Add an authenticated function to handle any ERC20 stuck into the contract for any reward received by any vault:

+    function withdrawToken(address _token) external requiresAuth {
+        // We can do that, as if the reward token is the same as the 
+        // asset token, then it will account for profit directly
+        if (address(ASSET_TOKEN) != _token) {
+            uint256 balance = IERC20(_token).balanceOf(address(this))
+            if(balance > 0){
+                IERC20(_token).safeTransfer(FEE_RECIPIENT,balance);
+            }        
+         }
+     }
