# Step 1: The Actor Model — Commutative Grid Dispatch

## The Problem We're Solving

Two control centers — one in California, one in New York — each manage a portion of the national grid. Both maintain a copy of the **grid balance**: a signed integer representing net generation in megawatts (positive = excess, negative = deficit).

Operators at either location can issue dispatch commands:
- **Dispatch up**: bring more generation online (+MW)
- **Curtail**: take generation offline (−MW)

Every dispatch command from either node must be seen by both nodes, so both copies of the grid balance stay in sync.

> **Why two copies?** Redundancy. If one control center loses network connectivity, it can still read the last-known balance and issue local dispatch decisions. This is the fundamental driver of distributed state.

---

## The Actor Approach

In the classical actor model, components communicate via **asynchronous message passing**. When a message arrives, a **reaction** (message handler) is invoked. The crucial property is: **the order in which messages from multiple sources are handled is not defined**.

Here is what our system looks like:

![DistibutedPowerGrid Actor](fig/DistibutedPowerGrid1_Actor.svg)


The squiggly arrows (`~>`) are **physical connections** in Lingua Franca: they use TCP for reliable in-order delivery on each link, but carry **no timestamp coordination** between links. Messages from California and New York may arrive at either grid manager in any order.

---

## The Code

See [`src/DistibutedPowerGrid1_Actor.lf`](src/DistibutedPowerGrid1_Actor.lf).

The core reactor is `SimpleGridManager`:

```lf
reactor SimpleGridManager {
  input in1: int   // commands arriving from local operator
  input in2: int   // commands arriving from remote operator
  output out: int  // current balance reported back to local operator

  state balance: int = 0

  reaction(in1, in2) -> out {=
    if (in1->is_present) {
        self->balance += in1->value;
        lf_print("Local command %+d MW -> balance now %d MW",
                 in1->value, self->balance);
    }
    if (in2->is_present) {
        self->balance += in2->value;
        lf_print("Remote command %+d MW -> balance now %d MW",
                 in2->value, self->balance);
    }
    lf_set(out, self->balance);
  =}
}
```

The top-level federated program wires everything together:

```lf
federated reactor {
    op1 = new GridInterface(...)   // California operator console
    op2 = new GridInterface(...)   // New York operator console
    gm1 = new SimpleGridManager()
    gm2 = new SimpleGridManager()

    op1.command ~> gm1.in1    // California commands -> California manager (local)
    op2.command ~> gm2.in2    // New York commands   -> New York manager (local)
    op1.command ~> gm2.in1    // California commands -> New York manager (remote)
    op2.command ~> gm1.in2    // New York commands   -> California manager (remote)

    gm1.out ~> op1.status
    gm2.out ~> op2.status
}
```

Each grid manager receives commands from **both** operators and keeps its own copy of the balance. The local operator console gets the balance back from its local manager.

---

## Why This Works — Sometimes

The operation `balance += value` has a special mathematical property: it is **associative and commutative**. It doesn't matter what order the additions happen — the final sum is always the same.

This means that even though `gm1` and `gm2` may process the same two commands in different orders, they will eventually agree on the same balance. This property is called **eventual consistency**.

More formally, this design satisfies **ACID 2.0** properties (Helland & Campbell):
- **A**ssociative: `(a + b) + c = a + (b + c)`
- **C**ommutative: `a + b = b + a`
- **I**dempotent: TCP guarantees exactly-once delivery, so each command is applied exactly once
- **D**istributed: state is maintained at multiple nodes

A datatype with these properties is called a **Conflict-Free Replicated Datatype (CRDT)** — one of the simplest CRDTs in existence.

---

## The Catch

This design would allow operators to curtail generation far below zero — a dangerous grid imbalance that could trip protective relays and cause a cascading blackout. 

Any **business logic** that enforces limits (e.g., "don't curtail if balance is already at its minimum threshold") breaks commutativity — and with it, our consistency guarantees.

That's what we explore next.

---

## Exercises

1. Trace through a scenario: California dispatches +100 MW at time 0, New York curtails −100 MW at time 1 ms. Show that both grid managers reach the same balance regardless of message arrival order.

2. What would happen if TCP delivery were *not* guaranteed? How would the ACID 2.0 / CRDT properties need to change?

3. Now consider the potential effect of network delays in this example. How would network delays affect the consistency of this example? [[Further Optional Reading]](https://arxiv.org/abs/2301.08906)

---

**Next:** [Step 2 — When Operations Are Non-Commutative](02-inconsistency.md)
