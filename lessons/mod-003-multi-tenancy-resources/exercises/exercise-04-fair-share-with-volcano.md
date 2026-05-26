# Exercise 04: Fair-Share Scheduling with Volcano

## Objective

Install Volcano; demonstrate gang scheduling + queue-based fair share.

## Tasks

1. Install Volcano via Helm.
2. Define 2 queues: `training` (capacity 6 GPUs) + `serving` (2 GPUs).
3. Submit a `Job` with `minAvailable: 4` GPUs to `training`. Confirm it waits for 4 GPUs to be free simultaneously.
4. Submit serving pods individually to `serving`. Confirm they schedule independently.
5. Demonstrate borrowing: with `training` idle, a serving burst can use 4+ GPUs temporarily.
6. Add a PriorityClass for serving; demonstrate it preempting a training pod when contended.

## Deliverable

`BENCHMARK.md` with measurements:
- Latency from job submission → all gang-pods running (vs default scheduler)
- Throughput of serving pods under contention vs uncontended
- Preemption: how long from preempt signal → restored serving pod
