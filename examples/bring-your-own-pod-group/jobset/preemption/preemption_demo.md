# Gang Preemption Demo

Demonstrates PodGroup-level gang preemption using JobSets with bring-your-own PodGroups.

## Prerequisites

- Kind cluster with 1 worker node (16 CPUs)
- PriorityClasses created:
  - `low-priority` (value: 1)
  - `high-priority` (value: 100000)

```bash
kubectl apply -f low-priority.yaml -f high-priority.yaml
```

## Setup

Each JobSet creates 4 pods (2 replicas x 2 completions) requesting 3 CPUs each (12 CPUs total). Only one JobSet can fit on the worker node at a time.

Both JobSets have a Workload and PodGroup with `disruptionMode: PodGroup`, which ensures the entire gang is preempted together rather than individual pods.

## Steps

### 1. Apply the low-priority JobSet

```bash
kubectl apply -f low-priority-jobset.yaml
```

Wait for all 4 pods to be running:

```bash
kubectl get pods -l jobset.sigs.k8s.io/jobset-name=lp-js -o wide
```

### 2. Apply the high-priority JobSet

```bash
kubectl apply -f high-priority-jobset.yaml
```

### 3. Observe preemption

The scheduler preempts the entire low-priority PodGroup as a gang to make room for the high-priority JobSet:

```bash
kubectl get pods -l 'jobset.sigs.k8s.io/jobset-name in (lp-js,hp-js)' -o wide
kubectl get events --field-selector reason=Preempted
```

## Expected Result

- All 4 low-priority pods are preempted together (gang preemption via `disruptionMode: PodGroup`)
- All 4 high-priority pods schedule and run
- Events show `Preempted by podgroup <id> on node cluster` for each low-priority pod

## Cleanup

```bash
kubectl delete jobset lp-js hp-js
kubectl delete podgroup lp-abc-workers-def hp-abc-workers-def
kubectl delete workload lp-abc hp-abc
kubectl delete priorityclass low-priority high-priority
```
