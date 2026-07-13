# Topology-Aware Gang Scheduling (`Job.spec.scheduling.constraints.topology`)

Demonstrates constraining a Job's gang to a single topology domain (e.g. a
rack) using the native `spec.scheduling.constraints` field.

## How it works

```yaml
spec:
  scheduling:
    policy:
      gang:
        minCount: 4
    constraints:
      topology:
      - key: topology.kubernetes.io/rack
```

All pods in the Job's pod group must land on nodes that share the same
value of the given node label. Different runs of the same Job may pick
different domain instances (e.g. rack1 vs rack2), but a single run will
never be split across domains.

> **Requires the `TopologyAwareWorkloadScheduling` feature gate.** Without
> it, `spec.scheduling.constraints` is silently dropped from the generated
> `PodGroup` and pods are scheduled without any topology constraint. Enable
> it (alongside `GenericWorkload` and `WorkloadWithJob`) on the
> kube-apiserver, kube-scheduler, and kube-controller-manager:
>
> ```
> --feature-gates=GenericWorkload=true,WorkloadWithJob=true,TopologyAwareWorkloadScheduling=true
> ```

## Prerequisites

Label your worker nodes so that at least two nodes share a rack value
(this example uses 4 workers split into 2 racks of 2):

```bash
kubectl label node <node-a> <node-b> topology.kubernetes.io/rack=rack1
kubectl label node <node-c> <node-d> topology.kubernetes.io/rack=rack2
```

## Files

| File | Purpose |
|------|---------|
| `topology-job.yaml` | A 4-pod gang Job constrained to a single `topology.kubernetes.io/rack` domain. |

## Usage

```bash
kubectl apply -f topology-job.yaml
kubectl get pods -l job-name=topology-job -o wide
```

Check which nodes the pods landed on and confirm they share the same rack
label:

```bash
kubectl get pods -l job-name=topology-job -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.nodeName}{"\n"}{end}'
kubectl get nodes -L topology.kubernetes.io/rack
```

Expected: all 4 pods land on nodes with the *same* rack label (e.g. all on
rack2), never split 2-and-2 across racks.

Inspect the auto-generated `PodGroup` to see the constraint that was
derived from the Job:

```bash
kubectl get podgroups -o yaml | grep -A3 schedulingConstraints
```

## Cleanup

```bash
kubectl delete -f topology-job.yaml
```
