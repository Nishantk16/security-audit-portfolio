# Accrued Interest Is Not Charged After a Full-Principal `debtWriteOff`

**Protocol:** Union Finance (union-v2-contracts)
**Contract:** `contracts/market/UToken.sol`
**Commit:** `632d7fb4f57e0b99355949bdf905ddda0fecd5bf`

---

## Status Update (Post-Research)

This finding was independently discovered on Day97 of a self-directed audit practice. Before submitting to Union's live Sherlock bug bounty, a check of Union's three prior Sherlock contests turned up an identical root-cause report already submitted by another researcher in the 2024 "Union Finance Update #2" contest (report "Bigsam - Interest Free Loans can be obtained from the Contract", sherlock-audit/2024-06-union-finance-update-2-judging#29).

That report was judged **Invalid** by the Sherlock lead judge (WangSecurity), including after a formal escalation, on the grounds that clearing only principal (and not interest) on `debtWriteOff` is the protocol's intended design — the borrower's loan is meant to be closed outright on write-off, and Union's own FAQ states stakers accept credit risk with no recourse if a borrower defaults ("this is the risk of credit").

**Conclusion:** the underlying observation below is technically correct (the `interest` field is genuinely left unresolved), but it does not qualify as a valid bug under Union's/Sherlock's judging standard, since it is accepted protocol design rather than a logic defect. This report is kept here as a record of the research process — confirming a hypothesis with a working PoC, then verifying it against prior judging history before spending a submission deposit — rather than as an active bounty submission.

---

## Summary

When a borrower's stake is written off via `UserManager.debtWriteOff()` → `UToken.debtWriteOff()`, the write-off logic zeroes out only the borrower's `principal` field. It never reads or adjusts the separate `interest` field on the same `BorrowSnapshot` struct. Immediately afterward, `calculatingInterest()` short-circuits to `0` whenever `principal == 0`, regardless of any value still held in `interest`. The practical effect is that any interest that had accrued on the loan up to the moment of write-off is not charged to the defaulting borrower and is not credited to `totalReserves` or `totalRedeemable`.

## Root Cause

`BorrowSnapshot` tracks principal and interest as two independent fields:

```solidity
struct BorrowSnapshot {
    uint256 principal;
    uint256 interest;
    uint256 interestIndex;
    uint256 lastRepay;
}
```

`getBorrowed()` reads only the `principal` field:

```solidity
function getBorrowed(address account) public view override returns (uint256) {
    return accountBorrows[account].principal;
}
```

`debtWriteOff()` uses exactly this value and touches nothing else:

```solidity
function debtWriteOff(address borrower, uint256 amount) external override whenNotPaused onlyUserManager {
    if (amount == 0) revert AmountZero();

    uint256 oldPrincipal = getBorrowed(borrower);
    uint256 repayAmount = amount > oldPrincipal ? oldPrincipal : amount;

    accountBorrows[borrower].principal = oldPrincipal - repayAmount;
    totalBorrows -= repayAmount;

    if (repayAmount == oldPrincipal) {
        accountBorrows[borrower].lastRepay = 0;
    }
}
```

Once `principal` is driven to `0`, `calculatingInterest()` returns `0` unconditionally:

```solidity
function calculatingInterest(address account) public view override returns (uint256) {
    BorrowSnapshot memory loan = accountBorrows[account];

    if (loan.principal == 0) {
        return 0;
    }
    ...
}
```

## Why This Was Judged Invalid

Per the Sherlock judging thread on the 2024 duplicate report:

> "It's the design of the protocol to not repay the interest here, since the result of this function is to make the borrower to not have any outstanding loans... Design decisions are not valid issues... Result: Invalid"

Union's own documentation reinforces this: `debtWriteOff` is documented as repaying the trove's **principal**, and the protocol FAQ states that if a borrower doesn't repay, the staker who vouched for them simply loses the staked amount — "this is the risk of credit," not a bug to be fixed.

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {TestUTokenBase} from "../uToken/TestUTokenBase.sol";
import {console2} from "forge-std/console2.sol";

contract TestDebtWriteOffInterestLoss is TestUTokenBase {
    function testInterestIsLostAfterFullDebtWriteOff() public {
        uint256 borrowAmount = 10 ether;

        vm.prank(ALICE);
        uToken.borrow(ALICE, borrowAmount);

        vm.warp(block.timestamp + 30 days);
        uToken.accrueInterest();

        uint256 accruedInterestBeforeWriteOff = uToken.calculatingInterest(ALICE);
        assertGt(accruedInterestBeforeWriteOff, 0, "setup invalid: no interest accrued");

        uint256 principalBeforeWriteOff = uToken.getBorrowed(ALICE);
        assertGt(principalBeforeWriteOff, 0, "setup invalid: principal already zero");

        vm.prank(address(userManagerMock));
        uToken.debtWriteOff(ALICE, principalBeforeWriteOff);

        uint256 accruedInterestAfterWriteOff = uToken.calculatingInterest(ALICE);
        uint256 totalOwedAfterWriteOff = uToken.borrowBalanceView(ALICE);

        assertEq(uToken.getBorrowed(ALICE), 0, "principal should be fully written off");
        assertEq(accruedInterestAfterWriteOff, 0, "interest silently reset to 0 (confirmed, but treated as intended design)");
        assertEq(totalOwedAfterWriteOff, 0, "borrower shows zero debt after write-off (confirmed, but treated as intended design)");
    }
}
```

**Test run output:**
