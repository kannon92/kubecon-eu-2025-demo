# Bring Your Own PodGroup

This example demonstrates how to manually create a `PodGroup` resource and associate it with a `Workload`, rather than letting the system generate one automatically. This is useful when you need explicit control over PodGroup configuration and lifecycle.

## Overview

In the default flow, creating a `Workload` with a `podGroupTemplate` automatically generates a corresponding `PodGroup`. In this example, you create the `PodGroup` yourself and link it back to the `Workload` via `podGroupTemplateRef`.

The example runs a 1000-pod parallel Job with gang scheduling — all 1000 pods must be schedulable before any of them are dispatched.

## Resources

| File | Resource | Description |
|------|----------|-------------|
| `workload.yaml` | `Workload` | Defines a workload with a pod group template requiring a gang `minCount` of 1000. |
| `podgroup-workload.yaml` | `PodGroup` | A manually created PodGroup that references the workload's pod group template. |
| `job-template.yaml` | `Job` | A parallel Job (1000 completions, indexed) whose pods reference the PodGroup via `schedulingGroup`. |

## Usage

1. **Create the Workload:**

   ```bash
   kubectl apply -f workload.yaml
   ```

2. **Create the PodGroup:**

   ```bash
   kubectl apply -f podgroup-workload.yaml
   ```

3. **Create the Job:**

   ```bash
   kubectl apply -f job-template.yaml
   ```

The Job's pods will only be scheduled once the gang scheduling requirement is met (all 1000 pods can be placed). Each pod runs a `busybox` container that sleeps for 10000 seconds, requesting 100m CPU and 100Mi memory.

## How It Works

- The `Workload` (`job-1`) declares a `podGroupTemplate` named `job-1-template` with a gang scheduling policy (`minCount: 1000`).
- The `PodGroup` (`job-1-template`) is created separately and links back to the workload using `podGroupTemplateRef`, specifying both the `workloadName` and `podGroupTemplateName`.
- The `Job` pods reference the PodGroup by setting `schedulingGroup.podGroupName: job-1-template` in the pod spec.

This three-resource wiring gives you full control over the PodGroup lifecycle while still integrating with the workload-aware scheduling system.
