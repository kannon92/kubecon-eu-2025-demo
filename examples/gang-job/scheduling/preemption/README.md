# Gang Preemption with Priority

Demonstrates priority-based **gang** preemption: when a higher-priority gang
needs room, the entire lower-priority gang is preempted together (not
pod-by-pod), thanks to `disruptionMode: All`.

## A note on `Job.spec.scheduling` and priority

`Job.spec.scheduling` (see the [`gang`](../gang) and
[`topology`](../topology) examples) is the simplest way to get workload-aware
scheduling for a plain Job — the controller auto-creates a `Workload` and
`PodGroup` for you. However, at the time of writing, the auto-derived
`PodGroup` is always created with `priority: 0` — the pod template's
`priorityClassName` is **not** propagated to it:

```
$ kubectl describe pod ...
Warning  FailedScheduling  ...  all pods in a single pod group should have
the same priority as the pod group's priority, got 1 and 0
```

Setting `priorityClassName` on a Job's pod template while also using
`spec.scheduling` therefore breaks scheduling entirely, regardless of
resource pressure, once the class resolves to a non-zero priority value.

You can reproduce this yourself with
[`native-broken/native-priority-gang-job.yaml`](native-broken/native-priority-gang-job.yaml)
— the "obvious" way to write a priority-aware native gang Job, kept here
deliberately because it fails:

```bash
kubectl apply -f 00-priorityclasses.yaml
kubectl apply -f native-broken/native-priority-gang-job.yaml
kubectl get pods -l job-name=native-priority-gang
# all 4 pods stay Pending forever, even though the cluster is otherwise idle
kubectl describe pod -l job-name=native-priority-gang | grep -A3 Events
kubectl delete -f native-broken/native-priority-gang-job.yaml -f 00-priorityclasses.yaml
```

Until that gap is closed, priority-based gang preemption is done by
hand-authoring the `Workload` and `PodGroup` (setting `priorityClassName`
and `disruptionMode` explicitly on the `PodGroup`) and pointing the Job's
pods at it with `template.spec.schedulingGroup.podGroupName` — the same
pattern used for JobSets in
[`../../bring-your-own-pod-group`](../../bring-your-own-pod-group). That's
exactly what this example does.

## Setup

Each Job has 4 pods (via required pod anti-affinity, one pod per node),
each requesting nearly a whole node's CPU. Only one Job's gang fits on the
4-node cluster at a time.

Both Jobs' `PodGroup`s have `disruptionMode: { all: {} }`, ensuring the
entire gang is preempted together rather than pod-by-pod.

## Files

| File | Resources | Description |
|------|-----------|--------------|
| `00-priorityclasses.yaml` | `PriorityClass` x2 | `low-priority` (value 1) and `high-priority` (value 100000). |
| `01-low-priority-job.yaml` | `Workload`, `PodGroup`, `Job` | 4-pod gang, one pod per node, `low-priority`. |
| `02-high-priority-job.yaml` | `Workload`, `PodGroup`, `Job` | Same footprint, `high-priority`. |

## Steps

### 1. Create the PriorityClasses

```bash
kubectl apply -f 00-priorityclasses.yaml
```

### 2. Apply the low-priority Job

```bash
kubectl apply -f 01-low-priority-job.yaml
```

Wait for all 4 pods to be running (one per node):

```bash
kubectl get pods -l job-name=low-priority-job -o wide
kubectl get podgroup low-priority-pg
```

### 3. Apply the high-priority Job

```bash
kubectl apply -f 02-high-priority-job.yaml
```

### 4. Observe gang preemption

```bash
kubectl get pods -l 'job-name in (low-priority-job,high-priority-job)' -o wide
kubectl get events --field-selector reason=Preempted
kubectl get podgroups
```

## Expected Result

- All 4 low-priority pods are preempted together (gang preemption via
  `disruptionMode: All` on the low-priority `PodGroup`).
- All 4 high-priority pods schedule and run in their place.
- Events show `Preempted by podgroup <id> on node cluster` for each
  low-priority pod.

## Cleanup

```bash
kubectl delete -f 02-high-priority-job.yaml
kubectl delete -f 01-low-priority-job.yaml
kubectl delete -f 00-priorityclasses.yaml
```
