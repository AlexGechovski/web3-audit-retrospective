# 🔍 ReserveLogic::_updateIndexes() assumes the utilization rate was constant the whole time when calculating the new borrows

## 📌 Summary  
_What was the issue?_  


    When calculating the new borrowing index we use latestBorrowingIndex(reserve) which is not updated when there are new borrows so it can be outdated. This means that the new borrowing index will be lower than it should, leading to a loss of profit for the protocol.

## ❌ Why I Missed It  
_What caused me to overlook this issue?_  
    Didn't take into account that the borrwing index is not updated when there are new borrows. 

## ✅ How I Can Avoid Missing This in the Future  
_What will I do differently next time?_  

   I should be carefull with important global variables when they are updated and when they should be and what that means for the protocol.

## 🔎 The Finding

ReserveLogic::_updateIndexes() assumes the utilization rate was constant the whole time when calculating the new borrows
Summary
ReserveLogic::_updateIndexes() is as follows:

function _updateIndexes(DataTypes.ReserveData storage reserve) internal {
    uint256 newBorrowingIndex = reserve.borrowingIndex;
    uint256 newTotalBorrows = reserve.totalBorrows;

    if (reserve.totalBorrows > 0) {
        newBorrowingIndex = latestBorrowingIndex(reserve);
        newTotalBorrows = newBorrowingIndex * (reserve.totalBorrows) / (reserve.borrowingIndex);
    ...
}
Note that when calculating the newBorrowingIndex, it calls latestBorrowingIndex(reserve), which uses reserve.currentBorrowingRate, which was calculated at the last time the index was updated. However, in the mean time, the borrows have grown, increasing the utilization rate, which has increased the borrowing rate, but the one used was the past one, which is incorrect.
To fix this, a better closed formula would be required, simultaneously calculating the new borrows as well as the current utilization rate, which depend on one another.

Root Cause
In ReserveLogic:155/156, new borrows use a stale utilization rate.

Internal Pre-conditions
None.

External Pre-conditions
None.

Attack Path
Users borrow from the lending pool and their interest will be incorrect, smaller than it should be.
Impact
Depending on utilization, the impact can be very significant, however, it is likely to stay relatively constrained due to daily utilization or similar.

PoC
Borrow interest depends on the utilization rate. The utilization rate grows with borrows, so it can not stay constant when calculating the new borrows over time.

Mitigation
Implement a closed formula that calculates the new borrows and utilization rate correctly.