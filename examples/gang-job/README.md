# Gang Job

This example demonstrates how workloads and pod groups are created automatically under the hood when you submit a standard Kubernetes Job — no manual `Workload` or `PodGroup` resources required.

## Overview

Unlike the [bring-your-own-pod-group](../bring-your-own-pod-group) example where you manually create `Workload` and `PodGroup` resources, this example submits a plain `Job`. The scheduling system automatically creates the corresponding `Workload` and `PodGroup` resources behind the scenes.

The example runs a 10-pod parallel Job without explicit gang scheduling configuration, showing the default behavior of the workload-aware scheduling system.

## Resources

| File | Resource | Description |
|------|----------|-------------|
| `gang-job.yaml` | `Job` | A parallel Job (10 completions, indexed) with no explicit gang scheduling configuration. |

## Usage

1. **Create the Job:**

   ```bash
   kubectl apply -f gang-job.yaml
   ```

2. **Observe the auto-created resources:**

   ```bash
   kubectl get workloads
   kubectl get podgroups
   ```

The Job's pods will be scheduled normally. Each pod runs a `busybox` container that sleeps for 10000 seconds, requesting 100m CPU and 100Mi memory.

## How It Works

- When the `Job` (`no-gang`) is created, the scheduling system automatically generates a `Workload` and `PodGroup` for it.
- No `schedulingGroup` is set on the pod spec, so the system handles the association automatically.
- This is the simplest integration path — you get workload-aware scheduling without manually wiring up any scheduling resources.

Compare this with the [bring-your-own-pod-group](../bring-your-own-pod-group) example to see the difference between automatic and manual `PodGroup` management.
