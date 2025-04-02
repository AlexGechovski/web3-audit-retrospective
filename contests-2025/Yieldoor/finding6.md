# 🔍 Finding Title

## 📌 Summary  

    Withdraw function does not take into account the idle capital already available, leading to withdrawing too much liquidity from the strategy.

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

   I didn't think about the fact that the idle capital already available should be taken into account when calculating the amount to withdraw.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

    I should be more careful with the calculations and check if there are any other variables that can affect the calculation of the amount to withdraw.
    
## 🔎 The Finding

Vault::withdraw() withdraws too much liquidity leading to idle capital and loss of fees
Summary
Vault::withdraw() withdraws from the lp position whenever the idle capital is not enough. However, it withdraws too much as it does not take into account that some of the idle capital is already available.

Root Cause
In Vault:98, it withdraws too many shares.

Internal Pre-conditions
None.

External Pre-conditions
None.

Attack Path
User withdraws 1000 shares, worth 1000 tokens. There already are 500 tokens idle, but the code will still withdraw all 1000 tokens, keeping 500 tokens idle.
Impact
Loss of fees as the liquidity stays idle.

PoC
function withdraw(uint256 shares, uint256 minAmount0, uint256 minAmount1)
    public
    returns (uint256 withdrawAmount0, uint256 withdrawAmount1)
{
    IStrategy(strategy).collectFees();

    (uint256 totalBalance0, uint256 totalBalance1) = IStrategy(strategy).balances();

    uint256 totalSupply = totalSupply();
    _burn(msg.sender, shares);

    withdrawAmount0 = totalBalance0 * shares / totalSupply;
    withdrawAmount1 = totalBalance1 * shares / totalSupply;

    (uint256 idle0, uint256 idle1) = IStrategy(strategy).idleBalances();

    if (idle0 < withdrawAmount0 || idle1 < withdrawAmount1) {
        // When withdrawing partial, there might be a few wei difference.
        (withdrawAmount0, withdrawAmount1) = IStrategy(strategy).withdrawPartial(shares, totalSupply);
    }
Mitigation
Reduce the shares to withdraw from the strategy by the liquidity already available (idle).