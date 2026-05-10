# Step 1: The Actor Model: Commutative Grid Dispatch

## The Problem We're Solving

Two control centers, one in California and one in New York, each manage a portion of the national grid. Both maintain a copy of the **grid balance**: a signed integer representing net generation in megawatts (positive = excess, negative = deficit).

Operators at either location can issue dispatch commands:
- **Dispatch up**: bring more generation online (+MW)
- **Curtail**: take generation offline (−MW)

Every dispatch command from either node must be seen by both nodes, so both copies of the grid balance stay in sync.

> **Why two copies?** Redundancy and availability. If one control center loses network connectivity, it can still read the last-known balance and issue local dispatch decisions. This is the fundamental driver of distributed state.

---

## The Actor Approach

In the classical actor model, components communicate via **asynchronous message passing**. When a message arrives, a **reaction** (message handler) is invoked. The crucial property is: **the order in which messages from multiple sources are handled is not defined**.

Here is what our system looks like:

![Step 1 actor model diagram](fig/Step1_Actor.svg)


The squiggly arrows (`~>`) are [**physical connections** in Lingua Franca](https://www.lf-lang.org/docs/writing-reactors/composing-reactors/#physical-connections). They still use TCP for reliable, in-order delivery on each individual link, but the receiver assigns the incoming message a logical timestamp based on its own physical clock (device's clock) rather than preserving the sender's logical timestamp. As a result, LF does not coordinate a single logical ordering across the California and New York links: messages from the two operators may arrive at either grid manager in either order.

---

## The Code

See [`src/Step1_Actor.lf`](src/Step1_Actor.lf).

The core reactor is `SimpleGridManager`:

```lf
reactor SimpleGridManager {
  input in1: int   // commands arriving from California
  input in2: int   // commands arriving from New York
  output out: int  // current balance reported back to local operator

  state balance: int = 0

  reaction(in1, in2) -> out {=
    if (in1->is_present) {
        self->balance += in1->value;
        lf_print("California command %+d MW -> balance now %d MW",
                 in1->value, self->balance);
    }
    if (in2->is_present) {
        self->balance += in2->value;
        lf_print("New York command %+d MW -> balance now %d MW",
                 in2->value, self->balance);
    }
    lf_set(out, self->balance);
  =}
}
```

The top-level federated program wires everything together. For the first exercise, the operator consoles are scripted with parameters, so you can change the trace without writing new reactors or timers:

```lf
federated reactor {
    gi1 = new ScriptedGridInterface(
        node_name="California",
        command_value=100,
        command_time=0 ms
    )
    gi2 = new ScriptedGridInterface(
        node_name="New York",
        command_value=-100,
        command_time=1 ms
    )
    gm1 = new SimpleGridManager(node_name="California manager")
    gm2 = new SimpleGridManager(node_name="New York manager")

    gi1.command ~> gm1.in1    // California commands -> California manager (local)
    gi2.command ~> gm2.in2    // New York commands   -> New York manager (local)
    gi1.command ~> gm2.in1    // California commands -> New York manager (remote)
    gi2.command ~> gm1.in2    // New York commands   -> California manager (remote)

    gm1.out ~> gi1.status
    gm2.out ~> gi2.status
}
```

Each grid manager receives commands from **both** operators and keeps its own copy of the balance. The local operator console gets the balance back from its local manager.

---

## Running Step 1

Compile the LF program with `lfc`:

```bash
lfc src/Step1_Actor.lf
```

Because this is a federated LF program, compilation generates a launcher under `bin/` named after the source file:

```bash
./bin/Step1_Actor
```

This launches the runtime infrastructure (RTI) and the four federates:

- `federate__gi1`: California grid interface
- `federate__gi2`: New York grid interface
- `federate__gm1`: California grid manager
- `federate__gm2`: New York grid manager

To see each federate in its own terminal pane, run the launcher with `--tmux`:

```bash
./bin/Step1_Actor --tmux
```

If `tmux` is not installed, install it first, for example with `brew install tmux` on macOS or `sudo apt-get install tmux` on Ubuntu.

Inside the tmux view, the top pane is the RTI and the other panes are the federates. The program has a built-in timeout, so it should finish on its own. To leave and close the entire tmux session after the run, press `Ctrl+B`, then `D`. If you need to stop a still-running federation, press `Ctrl+C` in the RTI pane, then detach with `Ctrl+B`, then `D`.

Example tmux run:

![Step 1 tmux run](assets/step1-actor-tmux.png)

In the screenshot, the managers receive the California and New York commands in different orders, but both managers end with balance `0 MW`.

---

## Why This Works, Sometimes

The operation `balance += value` has a special mathematical property: it is **associative and commutative**. It doesn't matter what order the additions happen; the final sum is always the same.

This means that even though `gm1` and `gm2` may process the same two commands in different orders, they will eventually agree on the same balance. This property is called **eventual consistency**.

More formally, this design satisfies **ACID 2.0** properties (Helland & Campbell):
- **A**ssociative: `(a + b) + c = a + (b + c)`
- **C**ommutative: `a + b = b + a`
- **I**dempotent: TCP guarantees exactly-once delivery, so each command is applied exactly once
- **D**istributed: state is maintained at multiple nodes

A datatype with these properties is called a [**Conflict-Free Replicated Datatype (CRDT)**](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type). This example is one of the simplest CRDTs in existence.

---

## The Catch

This design would allow operators to curtail generation far below zero, creating a dangerous grid imbalance that could trip protective relays and cause a cascading blackout.

Any **business logic** that enforces limits (e.g., "don't curtail if balance is already at its minimum threshold") breaks commutativity. That also breaks the consistency guarantee from this simple CRDT-style design.

That's what we explore next.

---

## Exercises

1. Modify `GridManager` to reject any command that would push `balance` below **−200 MW**, logging a warning instead. 

2. Run a scenario where both managers start at `balance = −150 MW`, California curtails −80 MW, and New York dispatches +100 MW simultaneously. What does each manager report?

3. Swap the message arrival order from Exercise 2. Do `gm1` and `gm2` agree on the final balance? Explain why the result differs and which ACID 2.0 property is violated.

---

**Next:** [Step 2: When Operations Are Non-Commutative](02-inconsistency.md)
