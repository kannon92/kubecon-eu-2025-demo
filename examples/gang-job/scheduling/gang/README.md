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
| `02-gang-job.yaml` | The same Job, but with `scheduling.policy.gang.minCount: 4` set. |

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

## Cleanup

```bash
kubectl delete -f 02-gang-job.yaml -f 00-filler.yaml --ignore-not-found
```
