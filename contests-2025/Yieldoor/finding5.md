Strategy::checkPoolActivity() incorrect check leads to vulnerable price

## 📌 Summary  
_What was the issue?_  
- Briefly describe the vulnerability.  
- Which function(s) were affected?  
- How severe was it (low, medium, high, critical)?  

    Incorrect timestamp check for when the cardinality is increased in the pool, because when the cardinality is increased, the timestamp is set to 1 for gas saving purposes. This means that the check will not revert when it should, leading to a vulnerable price observation.
    
    This is UniswapV3 specific oracle implementation.

    The check is:
    ```solidity
    if (timestamp == 0) {
        revert("timestamp 0");
    }
    ```
    but it should be:
    ```solidity
    if (timestamp == 1) {
        revert("timestamp 1");
    }
    ```


## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

   I haven't explored in depth the UniswapV3 oracle implementation, so I didn't know that the timestamp is set to 1 when the cardinality is increased.

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

    For future audits, I should explore more the used protocols and their implementation, especially if they are not the most common ones. I should also check the timestamp checks in the code, as they can be easily overlooked.

## 🔎 The Finding

Strategy::checkPoolActivity() incorrect check leads to vulnerable price
Summary
Strategy::checkPoolActivity() reverts if the timestamp is 0. However, this will never happen. What actually happens is the timestamp being 1, when the cardinality has been increased and the tick timestamp was set to 1 for gas saving purposes.

When this tick is hit in the loop, it means there were not enough observations to reach the lookAgo timestamp and it should revert because the price could not be validated. However, if the timestamp is 1, it may not revert depending on the value of the tick delta calculated, and as the timestamp is 1, it will return true with an invalid price observation.

Root Cause
In Strategy:322, the timestamp check is incorrect.

Internal Pre-conditions
None.

External Pre-conditions
None.

Attack Path
Twap would not have enough credibility and rebalancing would be impossible.
Attacker or user adds cardinality to the uniswap pool, which sets the next ticks timestamp to 1.
Rebalance goes through with an invalid price with not enough validation, leading to depositing liquidity at a bad price and causing losses for users.
Impact
Loss of funds due to depositing liquidity at an unfavourable price.

PoC
The following function is used when cardinality is increased in the pool.

    function grow(
        Observation[65535] storage self,
        uint16 current,
        uint16 next
    ) internal returns (uint16) {
        require(current > 0, 'I');
        // no-op if the passed next value isn't greater than the current next value
        if (next <= current) return current;
        // store in each slot to prevent fresh SSTOREs in swaps
        // this data will not be used because the initialized boolean is still false
        for (uint16 i = current; i < next; i++) self[i].blockTimestamp = 1;
        return next;
    }
Mitigation
            if (timestamp == 1) {
                revert("timestamp 1");
            }