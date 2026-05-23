# Lecture 02: Fair-Share Scheduling for ML

Kubernetes' default scheduler is FIFO + bin-packing. ML workloads need
gang scheduling + queue-aware preemption + GPU-aware bin-packing. That's why
Volcano + Apache Yunikorn exist.

## Gang scheduling

Distributed training jobs need all N pods running simultaneously, or no
pods. Default scheduler may start a few and leave the rest queued; the
training job stalls.

Volcano `Job` CRD:
```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata: { name: ddp-train }
spec:
  minAvailable: 4              # all 4 workers must be schedulable; else none scheduled
  schedulerName: volcano
  tasks:
    - replicas: 4
      template:
        spec:
          containers:
            - name: worker
              image: ghcr.io/me/trainer:0.5
              resources: { limits: { nvidia.com/gpu: 1 } }
```

## Fair-share + hierarchical queues

Yunikorn lets you express:

```yaml
queues:
  - name: ml-platform
    resources:
      max: { cpu: 200, memory: 800Gi, nvidia.com/gpu: 16 }
    queues:
      - name: training
        resources: { guaranteed: { nvidia.com/gpu: 8 } }
      - name: serving
        resources: { guaranteed: { nvidia.com/gpu: 4 } }
```

Each subqueue gets a guaranteed share; if `training` is idle, `serving` can
borrow its capacity, and vice versa.

## Preemption

Configure PriorityClass:
- `gpu-inference-critical` priority 1_000_000
- `gpu-training-batch` priority 100

Batch training pods can be preempted to make room for critical inference,
restarted automatically.

## Backfill

When a long training job needs 8 GPUs and only 6 are free, the scheduler
holds the queue. A backfill scheduler runs small short jobs in the meantime
without delaying the head-of-line job.

## When to bring in Volcano/Yunikorn

Default scheduler:
- Single-tenant clusters
- Mostly serving workloads
- Small teams

Volcano/Yunikorn:
- > 5 teams sharing GPUs
- Mix of long-running training + serving
- Need backfill + preemption + queue hierarchy

The migration cost is non-trivial. Don't bring it in before you need it.
