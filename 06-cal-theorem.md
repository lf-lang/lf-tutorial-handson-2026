# Step 6: The CAL Theorem: Fundamental Limits

## There Is No Perfect Distributed System

Through Steps 1-5, we have navigated a series of tradeoffs. Each improvement in consistency came at a cost in availability, latency assumptions, or complexity. This is not an accident of poor engineering; it is a fundamental theorem.

---

## The CAL Theorem

[**Lee et al. (2023)**](https://doi.org/10.1145/3609119) proved the following result for distributed cyber-physical systems:

> **CAL Theorem:** It is impossible to achieve consistency without paying a price in availability. The minimum price is proportional to the latencies in the system: network communication latency, computation overhead, and clock synchronization error.

This is a strengthening of the famous **CAP theorem** (Brewer, 2000), which says you cannot have all three of: Consistency, Availability, and Partition-tolerance. The CAL theorem goes further: even in a system with *no* partitions, apparent latency, including network delay, computation overhead, and clock synchronization error, introduces an unavoidable tradeoff.

For our grid:

| Term | Meaning |
|------|---------|
| **Consistency** | Both control nodes agree on the grid balance at every logical timestamp |
| **Availability** | How quickly an operator's command takes effect (low wait = high availability) |
| **Latency** | Network round-trip time + clock sync error + computation overhead |

The CAL theorem says that strong consistency requires enough waiting, or enough tolerated inconsistency, to cover apparent latency. There is no shortcut.

---

## The Design Space, Summarized

Here is a map of the designs from this tutorial against the consistency-availability tradeoff. Step 5 is shown as two paths because it deliberately gives different guarantees to different classes of commands:

```
CONSISTENCY
 strong  ▲  Step 4: conservative coordination
         │    Strong consistency; may wait forever if an upstream node stops.
         │
         │  Step 5 slow path: curtailments
         │    Same strong-consistency path as Step 4 for high-risk commands.
         │
         │             Step 3: finite maxwait
         │               Strong consistency only while apparent latency <= maxwait;
         │               otherwise the tardy handler is part of the design.
         │
eventual │                                      Step 5 fast path: dispatch-up
         │                                        Fast response with later reconciliation.
         │
         │                                      Step 1: commutative actor
         │                                        Eventual consistency and maximum availability.
         │
 none    │                                      Step 2: non-commutative actor
         │                                        High availability, but possible permanent divergence.
         │
         └────────────────────────────────────────────────────────► AVAILABILITY
            low / may block                                      high / responds fast
```

---

## Eventual Consistency as a Baseline

The coordinated designs in Steps 3-5 aim to maintain **eventual consistency** when their stated assumptions hold: if commands stop arriving and all messages are eventually delivered, the two grid managers eventually agree on the same balance. Logical timestamps and reliable delivery make this possible, but fault handling determines what happens when latency assumptions are violated.

Eventual consistency is the minimum acceptable requirement for our grid application. The question is always: how much *stronger* than eventual consistency do you need, and what are you willing to pay?

---

## Bounding Risk with Fault Handlers

In Step 5, we saw how fault handlers (tardy clauses) allow a system to proceed optimistically, processing commands with a potentially stale estimate while detecting and recovering from ordering violations after the fact.

This is the practical engineering answer to the CAL theorem:

1. **Choose a finite maxwait** that matches your network's typical latency (e.g., 30–100 ms).
2. **Write fault handlers** for the cases where actual latency exceeds maxwait and an earlier event has been handled.
3. **Tune the fault handlers** to implement business-logic recovery: log a warning, correct the estimate, alert operators, or take the fast path offline.
4. **Monitor the fault rate** over time. If faults are rare (say, < 1 per day), your maxwait is probably well-calibrated. If faults are frequent, increase maxwait or improve your network.

This gives you **bounded unavailability with manageable risk**, the practical goal that the original paper describes.

---

## What We Didn't Cover

This tutorial focused on the core consistency mechanisms. The paper also discusses:

- **Communication cycles**: feedback paths can make conservative CAL analysis too pessimistic, so the paper derives tighter processing offsets from the LF program structure.
- **After delays** (`a.out -> b.in after 10 ms`): a way to relax consistency by a controlled logical time offset, improving availability while preserving deterministic behavior.
- **Deadlines and fault handlers**: LF mechanisms for detecting and handling cases where latency assumptions no longer hold.

---

## Summary of the Design Journey

| Step | Design | Consistency | Availability | Key Mechanism |
|------|--------|-------------|--------------|---------------|
| 1 | Commutative actor | Eventual | Maximum | ACID 2.0 / CRDT |
| 2 | Non-commutative actor | None | Maximum | (broken) |
| 3 | Timestamped, finite maxwait | Strong (if maxwait ≥ latency) | Bounded (<= maxwait) | Logical timestamps |
| 4 | Chandy-Misra, maxwait=forever | Strong (always) | Unbounded on failure | Null messages |
| 5 | Hybrid fast/slow path | Strong for curtailments; eventual for dispatches | Bounded for dispatches | tardy fault handlers |

---

## Further Reading

- **Lee et al. (2023)**, "[Consistency vs. Availability in Distributed Cyber-Physical Systems](https://arxiv.org/abs/2301.08906)." The paper this tutorial is based on.
- **Lingua Franca documentation**: [lf-lang.org/docs](https://lf-lang.org/docs), especially the federated execution and decentralized coordination sections.
- **Brewer (2000)**, CAP theorem, the original impossibility result for distributed databases.
- **Shapiro et al. (2011)**, Conflict-Free Replicated Data Types, the CRDT framework that makes Step 1 work.
- **Chandy & Misra (1979)**, "Distributed Simulation," the conservative coordination algorithm from Step 4.

---

## Final Exercises

1. In `Step3_Timestamps.lf`, change one logical connection to `gi1.command -> gm1.in1 after 50 ms`. Rerun the simultaneous dispatch/curtailment scenario. Does consistency hold? How does the `after` delay change the operator wait time?

2. In `Step5_Hybrid.lf`, set `QuickDispatch` `@maxwait` to 200 ms and `GridManager` `@maxwait` to `forever`. Run a curtailment and a dispatch simultaneously. Which path handles each, and what do the two acknowledgement timestamps tell you about the consistency-availability tradeoff?

3. Add a third federate `gm3` to `Step4_Conservative.lf` (copy of `gm2`). Observe that `GridManager` with `maxwait = forever` still reaches the same balance on all three nodes. What additional null-message wiring did you need?
