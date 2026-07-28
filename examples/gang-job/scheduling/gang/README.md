# Native Gang Scheduling (`Job.spec.scheduling.policy.gang`)

Demonstrates all-or-nothing gang scheduling for a plain `Job`, using the
native `spec.scheduling` field — no hand-authored `Workload`/`PodGroup`
required.

## How it works

```yaml
spec:
  scheduling:
    policy:
      gang:
        minCount: 4   # all 4 pods must be schedulable together, or none are
```

When set, the Job controller automatically creates a `Workload` and
`PodGroup` behind the scenes with a `Gang` scheduling policy. The scheduler
will only admit pods from this Job once `minCount` of them can start at the
same time.

## Files

| File | Purpose |
|------|---------|
| `00-filler.yaml` | 4 filler Pods, one pinned per worker node, sized so that only 3 of the 4 nodes have room left for a 3-CPU pod. |
| `01-basic-job.yaml` | A plain Job (no `scheduling` field) with the same 4x3-CPU footprint, for comparison. |
| `02-gang-job.yaml` | The same Job, but with native gang scheduling enabled. |
| `03-gang-preemption-job.yaml` | A Kubernetes 1.37 native priority-aware gang Job; it preempts a lower-priority gang as a unit. |
| `04-default-gang-job.yaml` | A Kubernetes 1.37 native gang Job with `gang: {}` and no explicit `minCount`. |

## Usage

1. **Fill the cluster** so that only 3 of 4 nodes have room for a 3-CPU pod:

   ```bash
   kubectl apply -f 00-filler.yaml
   kubectl get pods -l app=filler -o wide
   ```

2. **Run the basic Job** and observe partial scheduling:

   ```bash
   kubectl apply -f 01-basic-job.yaml
   kubectl get pods -l job-name=basic-job -o wide
   ```

   Expected: 3 of 4 pods schedule and run immediately; the 4th stays
   `Pending` forever (standard pod-by-pod scheduling has no concept of the
   group).

   ```bash
   kubectl delete -f 01-basic-job.yaml
   ```

3. **Run the gang Job** and observe all-or-nothing behavior:

   ```bash
   kubectl apply -f 02-gang-job.yaml
   kubectl get pods -l job-name=gang-job -o wide
   kubectl get workloads,podgroups
   ```

   Expected: **all 4 pods stay `Pending`** — the scheduler refuses to admit
   any of them because only 3 of the 4 needed slots are available. The
   auto-created `PodGroup` shows `STATUS: Unschedulable`, and pod events show:

   ```
   pod group is unschedulable, minCount (4) cannot be satisfied: 3 scheduled, 0 remaining
   ```

4. **Free up capacity** to see the gang admit as a whole:

   ```bash
   kubectl delete -f 00-filler.yaml
   kubectl get pods -l job-name=gang-job -o wide
   ```

   All 4 pods should now transition to `Running` together.

## Default gang size

`04-default-gang-job.yaml` enables native gang scheduling without specifying
`minCount`, and enables gang-wide disruption with `disruptionMode: all`:

```yaml
scheduling:
  schedulingPolicy:
    gang: {}
  disruptionMode:
    all: {}
```

The Job API applies the default gang size for the Job, while disruption and
preemption operate on the gang as a unit. Apply it with:

```bash
kubectl apply -f 04-default-gang-job.yaml
kubectl get pods -l job-name=default-gang-job -o wide
kubectl get workload,podgroup
```

Delete it with:

```bash
kubectl delete -f 04-default-gang-job.yaml --ignore-not-found
```

## Gang scheduling with preemption

`03-gang-preemption-job.yaml` combines both features using the Kubernetes
1.37 native Job API. It sets `spec.scheduling.schedulingPolicy.gang.minCount:
4`, `spec.scheduling.disruptionMode: all`, and a non-zero
`priorityClassName`. In 1.37, the Job controller propagates the priority to
its generated `PodGroup`, so no explicit `Workload` or `PodGroup` is needed.

Run it with the priority-based preemption example as the lower-priority
victim gang:

```bash
kubectl apply -f ../preemption/00-priorityclasses.yaml
kubectl apply -f ../preemption/01-low-priority-job.yaml
# wait for the four low-priority pods to run
kubectl apply -f 03-gang-preemption-job.yaml
kubectl get pods -l 'job-name in (low-priority-job,gang-preempting-job)' -o wide
kubectl get podgroups
```

Expected: the four low-priority pods are preempted together and the
four-pod `gang-preempting-job` is admitted together. The low-priority
`PodGroup` must have `disruptionMode: all` for gang-wide victim preemption.
The high-priority `PriorityClass` is created by
`../preemption/00-priorityclasses.yaml`. This example requires Kubernetes
1.37 with the native Job scheduling feature enabled.

Cleanup:

```bash
kubectl delete -f 03-gang-preemption-job.yaml --ignore-not-found
kubectl delete -f ../preemption/01-low-priority-job.yaml --ignore-not-found
kubectl delete -f ../preemption/00-priorityclasses.yaml --ignore-not-found
```

## Cleanup

```bash
kubectl delete -f 02-gang-job.yaml -f 00-filler.yaml --ignore-not-found
```
