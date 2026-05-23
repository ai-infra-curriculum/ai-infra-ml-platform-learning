# Exercise 02: Point-in-Time Correctness

Demonstrate that without PIT joins, training data leaks future values.

Tasks:
1. Build a synthetic dataset where features evolve in time.
2. Train with a naive join (features as of "now"): measure training accuracy.
3. Train with a PIT join via Feast: measure training accuracy.
4. Compare offline accuracy to online accuracy at serving time.
5. Document the leakage: how much offline accuracy was artificially inflated.

Deliverable: `LEAKAGE_REPORT.md` showing measured delta + an explanation.
