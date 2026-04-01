# Step 5: Hybrid Design — Fast-Path for Safe Commands

## Exercise 1

> Suppose an operator issues a dispatch of +50 MW on the fast path. Their display shows a new balance of +50 MW. Two seconds later, the authoritative balance arrives and shows +150 MW (because a remote operator also dispatched +100 MW). Is this a problem? How would you handle the discrepancy in the UI?

## Exercise 2

> Design a `QuickCurtail` reactor that allows small curtailments (up to −50 MW) on a fast path. What STA and STAA values would you choose? What does the fault handler do when true_balance arrives late and shows the estimate was wrong?

## Exercise 3

> The fault handler for QuickDispatch logs a warning. In a real grid, what automated action might it trigger (e.g., alert, re-dispatch, trip breaker)?

---

**Next:** [Step 6 — The CAL Theorem: Fundamental Limits](06-cal-theorem.md)
