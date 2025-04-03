# 🔍 Finding Title

## 📌 Summary  
_What was the issue?_  

    Calclucation for shares can overflow when using low priced tokens. depositAmount * bal * totalSupply

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  

    I didn't think of scenario where there are low value tokens and didn't know it can overflow on a specific calculation

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

    Where there is calculation to check for overflows even in the low level if there isn't explicit casting and a math library isn't used.

## 🔎 The Finding

Vault::_calcDeposit() will overflow for low priced tokens
Summary
Vault::_calcDeposit() calculates the shares as depositAmount * bal * totalSupply in the numerator. If each of these quantities has 18 decimals, this is 1e54 of precision, having only around 1e23 left. Now, if bal and totalSupply are for example 1e9 each, this leaves 1e5 left. Thus, any deposit exceeding 100_000 will revert. If the token is valued for example 0.01 USD, this is very likely to happen.
As deposits failling will make users miss out on yield and favorable entry prices in the pool, it is time sensitive.

Root Cause
In Vault:149, there is an overflow risk.

Internal Pre-conditions
Token is low valued such as 0.1 or 0.01 USD.
External Pre-conditions
None.

Attack Path
User deposits in the vault but reverts, missing out on yield and a certain entry price in the pool.
Impact
User loses yield and may get worse slippage / price afterwards. Additionally, until someone withdraws it may be hard to deposit. They may split their deposits in smaller ones but this has other implications. Given enough balance and supply of a low valued token it may not be possible to deposit at all.

PoC
See above.

Mitigation
Implement some precision downscaling in between calculations.