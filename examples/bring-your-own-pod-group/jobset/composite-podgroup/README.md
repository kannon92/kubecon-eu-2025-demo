# Replicated training JobSet with CompositePodGroups

This example models the [replicated training jobs under JobSet use case in
KEP-6012](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/6012-composite-podgroup-api#replicated-training-jobs-under-jobset).

The JobSet contains two replicated stages:

* **Data and model initialization**: two one-pod replicated Jobs for each
  activity. The stage uses a `Basic` CompositePodGroup policy, so downloaders
  and initializers can be admitted independently.
* **Trainer**: two replicated MPI launcher Jobs and two replicated worker Jobs
  with three workers each. The two launchers and six workers share a `PodGroup`
  with `minCount: 8`, giving the trainer a strict gang.

Both stages are children of `replicated-training-root`. The root is `Basic`:
the two stages run sequentially, since the trainer stage's pods aren't
created until the data stage completes (see `dependsOn` below), so a `Gang`
policy requiring both stages ready at once would deadlock admission —
data-stage's pods would wait on trainer-stage, which can't exist without
data-stage completing first. `disruptionMode: all` on every CompositePodGroup
still demonstrates the coordinated preemption/disruption fate-sharing
described by the KEP, independent of the root's admission policy.

The JobSet also uses `dependsOn` as an execution policy: `trainer-launcher`
and `trainer` each declare `dependsOn: [{name: model-initializer, status:
Complete}]`, so the trainer stage's ReplicatedJobs aren't created until the
data stage finishes. `data-downloader` and `model-initializer` have no
`dependsOn` between them and stay parallel, matching their `Basic`
CompositePodGroup policy. `trainer-launcher` and `trainer` intentionally do
not depend on each other, since they share one 8-pod gang PodGroup and must
be created together — depending on each other's readiness would deadlock
gang admission. Note that `dependsOn` currently supports only one dependency
per ReplicatedJob, so this models "wait for the data stage" via
`model-initializer` alone rather than both data-stage jobs.

## Group hierarchy

```text
replicated-training-root (CompositePodGroup, Basic)
├── replicated-training-data (CompositePodGroup, Basic)
│   ├── replicated-training-data-downloaders (PodGroup, Gang minCount=2)
│   └── replicated-training-model-initializers (PodGroup, Gang minCount=2)
└── replicated-training-trainer (CompositePodGroup, Gang minGroupCount=1)
    └── replicated-training-trainer-workers (PodGroup, Gang minCount=8)
```

## Apply

This example assumes a Kubernetes distribution with the KEP-6012
`CompositePodGroup` API and the required scheduling feature gates enabled. It
also assumes the JobSet CRDs and controller are installed.

```sh
kubectl apply -f workload.yaml
kubectl apply -f jobset.yaml
```

The `Workload`, CPGs, and leaf `PodGroups` are included as separate YAML
objects in `workload.yaml` to make the hierarchy visible. In a production
JobSet integration, the JobSet controller (or an integration controller) would
normally create the runtime groups from the Workload templates. This example
uses the bring-your-own-PodGroup pattern, so the runtime groups are supplied
explicitly and the JobSet's `schedulingGroup.podGroupName` fields point at the
leaf PodGroups.

Inspect the result with:

```sh
kubectl get workload,compositepodgroup,podgroup,jobset,pods
```

The names in `workload.yaml` intentionally match the names referenced by the
JobSet. If the JobSet controller creates namespaced resources with generated
names in your deployment, update the `podGroupName` values in
`jobset.yaml` accordingly.
