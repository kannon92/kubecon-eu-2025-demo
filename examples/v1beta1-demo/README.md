# Beta workload-aware scheduling examples

These examples use the native `scheduling.k8s.io/v1beta1` APIs, not a scheduler
extension or a CRD.

## Prerequisites

Get a Kubernetes cluster with the `scheduling.k8s.io/v1beta1` APIs enabled.

Run the following commands from this directory:

```sh
kubectl apply -f namespace.yaml
```

Each pod requests 100m CPU and 32Mi memory,
runs for five minutes, and has a 200m CPU / 64Mi memory limit. Jobs have a
15-minute deadline, including time spent waiting to schedule.

## 1. A three-member gang

```sh
kubectl apply -f gang.yaml
kubectl -n scheduling-beta-examples wait podgroup/gang \
  --for=condition=PodGroupInitiallyScheduled --timeout=180s
kubectl -n scheduling-beta-examples get pods -l scheduling-example=gang -o wide
```

Expected: all three pods are assigned to nodes once the scheduler can admit the
three-member gang. The `PodGroupInitiallyScheduled` condition becomes `True`.
Gang scheduling coordinates admission/placement, not simultaneous container
startup; image pulls and startup times can differ. This example does not require
members to use the same node.

## 2. Hold two pods until the third member exists

```sh
kubectl apply -f wait-for-members.yaml
kubectl -n scheduling-beta-examples get pods \
  -l scheduling-example=wait-for-members -o wide --watch
```

Expected: two pods exist but remain Pending **without a node assignment**, because
the PodGroup requires three members and the Job initially has `parallelism: 2`.
Stop the watch with Ctrl-C, then release the gang:

```sh
kubectl -n scheduling-beta-examples patch job wait-for-members --type=merge \
  -p '{"spec":{"parallelism":3}}'
kubectl -n scheduling-beta-examples wait podgroup/wait-for-members \
  --for=condition=PodGroupInitiallyScheduled --timeout=180s
kubectl -n scheduling-beta-examples get pods \
  -l scheduling-example=wait-for-members -o wide
```

Expected: a third pod is created and all three receive node assignments when
capacity is available. Release the gang before the Job's 15-minute deadline.
If the first two pods get node assignments before the third exists, verify that
`spec.schedulingGroup.podGroupName` survived admission:

```sh
kubectl -n scheduling-beta-examples get pods \
  -l scheduling-example=wait-for-members \
  -o custom-columns='NAME:.metadata.name,GROUP:.spec.schedulingGroup.podGroupName,NODE:.spec.nodeName'
```

## 3. Workload template with same-node placement

```sh
kubectl apply -f workload-topology.yaml
kubectl -n scheduling-beta-examples get workload workload-topology -o yaml
kubectl -n scheduling-beta-examples wait podgroup/workload-topology-workers \
  --for=condition=PodGroupInitiallyScheduled --timeout=180s
kubectl -n scheduling-beta-examples get pods \
  -l scheduling-example=workload-topology -o wide
```

Expected: both pods are scheduled onto the **same node** because the topology
key is `kubernetes.io/hostname`. One node must have room for both pods' combined
requests (200m CPU / 64Mi memory), in addition to existing workloads.

The Workload defines a `podGroupTemplates` entry. The manifest explicitly creates
a PodGroup with matching policy/constraints and a `workloadRef` identifying that
template. The Job's pods reference the PodGroup via `spec.schedulingGroup`.
Creating a Workload alone does **not** create PodGroups or pods, and `workloadRef`
does not automatically copy template fields or establish garbage-collection
ownership.

These examples intentionally do not use `Job.spec.scheduling`, which the current
schema marks as requiring the separate alpha `WorkloadWithJob` gate. A plain Job
creates the pods; the explicitly created PodGroup provides the gang policy.

## 4. LeaderWorkerSet and JobSet gang scheduling

These examples let the controller create the Workload and PodGroups and set each
pod's `spec.schedulingGroup`; you only set `spec.scheduling` on the parent object.
Install the controllers with their alpha gates enabled:

```sh
helm upgrade --install lws oci://registry.k8s.io/lws/charts/lws --version 1.0.0 \
  -n lws-system --create-namespace -f lws-values.yaml
helm upgrade --install jobset oci://registry.k8s.io/jobset/charts/jobset --version 0.13.0 \
  -n jobset-system --create-namespace -f jobset-values.yaml
```

LWS needs both `featureGates.WorkloadAwareScheduling=true` and
`gangSchedulingManagement.schedulerProvider=kubernetes`; without the provider its
webhook rejects `spec.scheduling`. JobSet needs
`controller.featureGates.JobSetWorkloadAwareSchedulingAPI=true`.

| File | Level | PodGroups created |
|---|---|---|
| `lws-gang.yaml` | LWS replica (`scheduling: {}`) | one per replica, `minCount: 3` (leader + 2 workers) |
| `jobset-level1-jobset.yaml` | 1: whole JobSet | one, `minCount: 5` |
| `jobset-level2-replicatedjob.yaml` | 2: per ReplicatedJob | launcher `minCount: 1`, workers `minCount: 4` |
| `jobset-level3-job.yaml` | 3: per Job | launcher `minCount: 1`, one per worker Job `minCount: 2` (x3) |

```sh
kubectl apply -f lws-gang.yaml -f jobset-level1-jobset.yaml \
  -f jobset-level2-replicatedjob.yaml -f jobset-level3-job.yaml
kubectl -n scheduling-beta-examples get workloads
kubectl -n scheduling-beta-examples get podgroups -o custom-columns='NAME:.metadata.name,MIN:.spec.schedulingPolicy.gang.minCount,SCHEDULED:.status.conditions[?(@.type=="PodGroupInitiallyScheduled")].status'
kubectl -n scheduling-beta-examples get pods \
  -l 'scheduling-example in (lws-gang,jobset-level1,jobset-level2,jobset-level3)' \
  -o custom-columns='NAME:.metadata.name,GROUP:.spec.schedulingGroup.podGroupName,NODE:.spec.nodeName'
```

JobSet pods sleep for two minutes and complete. The LWS pods run until deleted:
`kubectl delete -f lws-gang.yaml`.

## 5. DisaggregatedSet (prefill/decode) gang scheduling

The DisaggregatedSet CRD and controller ship with the LWS v1.0.0 chart. The
controller creates one LeaderWorkerSet per role. It also copies each role's
`spec.scheduling` to that LWS, so every leader+workers group becomes its own gang.

| Role | Replicas x size | PodGroups created |
|---|---|---|
| `prefill` | 2 x 2 | two, `minCount: 2` each |
| `decode` | 1 x 3 | one, `minCount: 3` |

```sh
kubectl apply -f disaggregatedset-gang.yaml
kubectl -n scheduling-beta-examples get dset dset-gang
kubectl -n scheduling-beta-examples get lws   # dset-gang-0-<hash>-prefill and -decode
kubectl -n scheduling-beta-examples get podgroups -o custom-columns='NAME:.metadata.name,MIN:.spec.schedulingPolicy.gang.minCount,SCHEDULED:.status.conditions[?(@.type=="PodGroupInitiallyScheduled")].status'
kubectl -n scheduling-beta-examples get pods -l scheduling-example=dset-gang \
  -o custom-columns='NAME:.metadata.name,GROUP:.spec.schedulingGroup.podGroupName,NODE:.spec.nodeName'
```

Observed on this cluster: `Available=True`, three PodGroups (prefill x2
`minCount: 2`, decode `minCount: 3`), all `PodGroupInitiallyScheduled=True`, and
all 7 pods Running with `schedulingGroup` set. Child names include a slice index
and a revision hash (for example `dset-gang-0-de6e074f-decode`). The
chart's `enableDisaggregatedSet` value only adds the validating webhook and user
ClusterRoles, so it was not needed for this run. Pods run until deleted:
`kubectl delete -f disaggregatedset-gang.yaml`.

## Inspect, finish, and clean up

```sh
kubectl -n scheduling-beta-examples get jobs,pods,podgroups,workloads
kubectl -n scheduling-beta-examples get podgroups -o yaml
kubectl -n scheduling-beta-examples get events --sort-by=.lastTimestamp
kubectl -n scheduling-beta-examples wait --for=condition=complete job --all --timeout=10m
```

`Pending` with no node assignment is a scheduling observation; `ImagePullBackOff`
or `CreateContainerConfigError` after assignment is an image/runtime issue.
`PodGroupInitiallyScheduled=True` records initial admission, not Job completion,
and does not reset if members are later evicted. To repeat a test, delete its
manifest and recreate it so the PodGroup is fresh:

```sh
kubectl delete -f wait-for-members.yaml
kubectl apply -f wait-for-members.yaml
```

Remove everything created by these examples (including all resources in their
dedicated namespace):

```sh
kubectl delete -f namespace.yaml
```

## Validation performed

All YAML documents passed server-side create dry-run with strict field validation.
Namespaced resources were dry-run in the existing `default` namespace, and the
Job responses were checked to ensure `schedulingGroup` was preserved. Beta
resources were also checked against the cluster's OpenAPI schema. Dry-run did not
persist any resources.

The examples were subsequently applied after the control-plane rollout completed.
Observed: all three `gang` pods Running; both `workload-topology` pods Running on
the same node; both `wait-for-members` pods Pending without node assignments.
The parallelism-increase/release step and Job completion have not yet been verified.
