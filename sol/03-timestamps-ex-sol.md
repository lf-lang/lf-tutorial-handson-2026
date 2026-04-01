# Step 3: Logical Timestamps — Ordering Events Across the Grid

## Exercise 1

> With `STA = 100 ms` and cross-continental latency of 75 ms, what is the maximum clock synchronization error you can tolerate and still guarantee correct ordering?

## Exercise 2

> If you reduced STA to 20 ms to improve responsiveness, what would happen if a message arrived 30 ms late? What would the LF runtime do (hint: see the `fault handler` concept)?

## Exercise 3

> Revisit the inconsistency scenario from Step 2 (balance = −150 MW, simultaneous curtail and dispatch). Trace through the execution with timestamps. Do both grid managers reach the same balance?

---

**Next:** [Step 4 — Conservative Coordination with Null Messages](04-conservative.md)
