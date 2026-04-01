# Step 6: The CAL Theorem — Fundamental Limits: Final Exercises

## Exercise 1

> **Full scenario trace**: Starting from balance = −100 MW at both nodes, trace the complete execution of Step 5's hybrid design through: (a) California dispatches +200 MW, (b) New York curtails −150 MW (simultaneous), (c) California curtails −80 MW (2 seconds later). Which commands use the fast path? What does each GridManager report after each event?

## Exercise 2

> **STA calibration**: You deploy the Step 3 design between London and Tokyo (network latency ~250 ms, NTP error ~50 ms). What STA should you choose? What is the resulting unavailability (operator wait time) for curtailment commands?

## Exercise 3

> **Quorum design**: In the paper, Lee suggests using a quorum of nodes rather than unanimous agreement for determining the "true balance." How would you modify Step 5's `GridManager` to require agreement from 2 out of 3 nodes instead of 2 out of 2? What does the fault handler do when a node disagrees with the quorum?

## Exercise 4

> **Real-world mapping**: Map the design from Step 5 onto a real grid control architecture. What does `GridServer` correspond to? What does `GridManager` correspond to? What role does the `QuickDispatch` fast path play in existing SCADA/EMS systems?
