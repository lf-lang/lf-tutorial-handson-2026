# Step 4: Conservative Coordination — Chandy-Misra Null Messages

## The Problem with Finite STA

In Step 3, we set `STA = 100 ms`. This means:

- The grid manager waits 100 ms after logical time `t` before processing a message at `t`.
- This is safe *if and only if* every remote message arrives within 100 ms of its timestamp.

But what if the link between California and New York goes down for 5 minutes? Or a router becomes congested and introduces 500 ms of latency? The `STA = 100 ms` assumption is violated. The LF runtime will invoke the **fault handler** — but we need to have designed one.

A more principled approach avoids making assumptions about latency altogether.

---

## Conservative Coordination: Wait Until You Know

Instead of guessing that all messages will arrive within some time budget, the **conservative approach** says:

> A node must not process a message at logical time `t` until it has proof that no earlier message is coming from any connected node.

This is the Chandy-Misra algorithm (1979), originally developed for distributed simulation. LF supports it via `STA = forever`:

```lf
reactor GridManager(STA: time = forever) {
    // ...
}
```

With `STA = forever`, the grid manager will wait **indefinitely** — until it receives positive evidence from the remote node that no message with timestamp ≤ `t` is coming.

How does that evidence arrive? That's where **null messages** come in.

---

## Null Messages

In our grid, commands are sent only when operators issue dispatches or curtailments. If California issues no commands for 10 minutes, New York's manager has no evidence about California's timeline — and blocks forever under `STA = forever`.

The fix: California's node sends **null messages** periodically. A null message says:

> "I have no real command at this timestamp, but here is my current timestamp. You may safely advance past it."

In the grid context, a null message with value `0` means "no change in dispatch this QuickDispatchsecond." It is a heartbeat that lets the remote manager advance logical time even during quiet periods.

```lf
reactor GridServer(null_message_period: time = 1 s) {
    input  in: int
    output received: int
    timer  t(0, null_message_period)

    reaction(w.received, t) -> received {=
        if (w.received->is_present) {
            lf_set(received, w.received->value);  // real command
        } else {
            lf_set(received, 0);  // null message: no dispatch this tick
        }
    =}
}
```

The timer fires every second. If no real command has arrived, a `0` is forwarded — which the grid manager interprets as "no change." This lets the grid manager advance its logical time by at least 1 second per second, bounding the wait.

---

## Guarantees and Costs

**What you get:**
- **Strong consistency**: both grid managers agree on the balance at every logical timestamp. No STA violations, ever.
- **Bounded wait**: the maximum wait is ~1 second (the null message period) plus network latency.

**What you give up:**
- **Availability under failure**: if the California `GridServer` crashes, it stops sending null messages. New York's manager blocks indefinitely — it cannot advance time because it has no proof that California won't send a message.
- **Network overhead**: null messages are sent every second even when there are no real commands. For a grid with hundreds of nodes, this multiplies.

This is the CAL theorem in action: **strong consistency costs availability when network behavior is unbounded**.

---

## Code

See [`src/DistibutedPowerGrid4_ChandyMisra.lf`](src/DistibutedPowerGrid4_ChandyMisra.lf). And here is what our system looks like:

And here is what our system looks like:

![DistibutedPowerGrid4 Chandy & Misra](fig/DistibutedPowerGrid4_ChandyMisra.svg)

Key changes from Step 3:
- `GridInterface` is wrapped in `GridServer`, which adds a null-message timer.
- `GridManager` has `STA = forever`.
- Logical connections (`->`) are retained.

---

## Exercises

1. Trace through a scenario where California's GridServer crashes at time T = 30 s. What happens to New York's grid manager? What happens to California's grid manager?

2. If we reduce the null message period from 1 s to 100 ms, how does this affect (a) wait time, (b) network overhead, and (c) resilience to node failure?

3. Could we use `STA = forever` with *no* null messages at all? Under what circumstances would the system make progress?

---

**Next:** [Step 5 — Hybrid Design: Fast-Path for Safe Commands](05-hybrid.md)
