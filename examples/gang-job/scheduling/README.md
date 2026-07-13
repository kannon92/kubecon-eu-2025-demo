# Native Job Scheduling (`Job.spec.scheduling`)

These examples explore the alpha `spec.scheduling` field on the native
`batch/v1` Job API, which lets a plain Job opt into workload-aware
scheduling (gang scheduling, topology constraints, disruption modes) without
requiring users to hand-author `Workload`/`PodGroup` objects.

```
$ kubectl explain job.spec.scheduling
FIELD: scheduling <JobSchedulingConfiguration>

DESCRIPTION:
    scheduling defines the Workload-aware Scheduling configuration for this
    Job. When set, it specifies the scheduling policy (basic or gang),
    topology constraints, disruption mode, and shared resource claims. When
    omitted, the Job defaults to the basic scheduling policy, which behaves
    as standard pod-by-pod scheduling. This field is alpha-level and
    requires the WorkloadWithJob feature gate. This field is immutable,
    including whether it is set at all, only policy.gang.minCount may be
    changed after creation.
```

`JobSchedulingConfiguration` fields:

| Field | Purpose |
|-------|---------|
| `policy.basic` / `policy.gang.minCount` | Standard pod-by-pod scheduling, or all-or-nothing gang scheduling. Exactly one must be set. |
| `constraints.topology[].key` | Node label key defining a topology domain all pods in the group must share. Requires the `TopologyAwareWorkloadScheduling` feature gate. |
| `disruptionMode.single` / `disruptionMode.all` | Whether pods can be disrupted (e.g. preempted) independently, or must be disrupted together as a gang. |
| `resourceClaims` | Dynamic resource claims shared across pods in the group. |

Prerequisites for these examples: `GenericWorkload=true,WorkloadWithJob=true`
feature gates, plus `TopologyAwareWorkloadScheduling=true` for the
[`topology`](topology) example, enabled on the kube-apiserver,
kube-scheduler, and kube-controller-manager.

## Examples

| Directory | Demonstrates |
|-----------|--------------|
| [`gang/`](gang) | `scheduling.policy.gang.minCount` — all-or-nothing scheduling for a Job's pods, contrasted with plain pod-by-pod scheduling. |
| [`topology/`](topology) | `scheduling.constraints.topology` — keeping a Job's whole gang within a single topology domain (e.g. a rack). |
| [`preemption/`](preemption) | Priority-based **gang** preemption (`disruptionMode: All`), including a known current gap in the native path (see that README) and the hand-authored `Workload`/`PodGroup` workaround. |

## Relationship to other examples in this repo

- [`../gang-job`](..) shows a Job with **no** explicit `spec.scheduling`,
  relying entirely on defaults (basic scheduling with an auto-created
  `Workload`/`PodGroup`).
- [`../../bring-your-own-pod-group`](../../bring-your-own-pod-group) shows
  the fully manual path: hand-authoring `Workload` and `PodGroup` objects
  yourself and pointing pods (Job or JobSet) at them via
  `template.spec.schedulingGroup`. That path is still required today for
  scenarios like priority-based gang preemption, where the native
  `spec.scheduling` field doesn't yet propagate everything needed — see
  [`preemption/README.md`](preemption/README.md) for details.
