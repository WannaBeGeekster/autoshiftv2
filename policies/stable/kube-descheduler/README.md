# kube-descheduler AutoShift Policy

## Overview

Installs the Kube Descheduler Operator and manages its `KubeDescheduler` instance. The descheduler
evicts pods so that the scheduler can place them again on less loaded nodes. With Red Hat OpenShift
Virtualization it live migrates virtual machines instead, through the `KubeVirtRelieveAndMigrate`
profile.

Enable it with `kube-descheduler: 'true'`. With no further configuration it runs the
`AffinityAndTaints` profile in `Predictive` mode, which logs what it would evict and evicts nothing.

## Policies

| Policy | Placement | Purpose |
|---|---|---|
| `policy-kube-descheduler-operator-install` | `kube-descheduler` | Namespace `openshift-kube-descheduler-operator` and the operator subscription (OwnNamespace) |
| `policy-kube-descheduler-config` | `kube-descheduler` | `KubeDescheduler` named `cluster`, from `config.kubeDescheduler`. Waits on the operator install |
| `policy-kube-descheduler-psi` | `kube-descheduler` and `kube-descheduler-psi` | `MachineConfig` that adds the `psi=1` kernel argument |
| `policy-kube-descheduler-test` | `kube-descheduler` | Inform only. NonCompliant while the operator reports `TargetConfigControllerDegraded` |

The operator reconciles only the instance named `cluster` in its own namespace, which is why the
policy manages exactly that one.

## Configuration

Settings come from `config.kubeDescheduler`:

```yaml
config:
  kubeDescheduler:
    profiles:
      - KubeVirtRelieveAndMigrate
    mode: Predictive                    # Predictive simulates; Automatic evicts
    deschedulingIntervalSeconds: 60
    profileCustomizations:
      devEnableSoftTainter: true
      devDeviationThresholds: AsymmetricLow
      devActualUtilizationProfile: PrometheusCPUMemoryCombinedProfile
    evictionLimits:
      node: 2
      total: 5
```

| Key | Default | Notes |
|---|---|---|
| `profiles` | `[AffinityAndTaints]` | Profile names come from the operator. `KubeVirtRelieveAndMigrate` cannot be combined with `LongLifecycle` |
| `mode` | `Predictive` | Nothing is evicted until this is `Automatic` |
| `deschedulingIntervalSeconds` | `3600` | |
| `profileCustomizations` | not set | Passed through unchanged, so new fields need no policy change |
| `evictionLimits` | not set | Passed through unchanged |

The instance is applied with `mustonlyhave`, so a key removed from the configuration is removed from
the instance as well.

Start in `Predictive` mode. Read the descheduler logs in `openshift-kube-descheduler-operator` to see
what it would evict, then switch the clusterset to `Automatic`.

## Virtual machines

The `KubeVirtRelieveAndMigrate` profile rebalances virtual machines by live migration, based on
actual CPU and memory load. It needs two things beyond the profile:

1. **Pressure Stall Information (PSI) on every node that runs virtual machines.** Set
   `kube-descheduler-psi: 'true'`, and `policy-kube-descheduler-psi` adds the `psi=1` kernel
   argument. Set `kube-descheduler-psi-role` to the machine config pool of those nodes: `worker` by
   default, or `master` on a compact or single-node cluster, where the worker pool is empty. Until
   every node has PSI, the operator reports `TargetConfigControllerDegraded` and
   `policy-kube-descheduler-test` is NonCompliant.
2. **Eviction limits that match the migration limits.** Set `evictionLimits.node` to the
   `HyperConverged` `parallelOutboundMigrationsPerNode` and `evictionLimits.total` to
   `parallelMigrationsPerCluster`, so that the descheduler never requests more migrations than
   Red Hat OpenShift Virtualization runs at once.

> [!WARNING]
> A new kernel argument restarts every node in the target pool, one at a time, and each node must
> drain first. Enable `kube-descheduler-psi` in a maintenance window. On a compact cluster the
> `master` pool holds the control plane, so every control plane node restarts.

Before enabling it, check that a node drain cannot stall:

- A single-instance database protected by a `PodDisruptionBudget`, such as a CloudNativePG cluster
  with one instance, blocks the drain of its node. Give it a second instance on another node.
- The remaining nodes must hold the workload of the node being drained, in pods per node and in
  requested CPU and memory.
- Virtual machines must be live migratable.

To keep a virtual machine where it is, annotate it with
`descheduler.alpha.kubernetes.io/prefer-no-eviction`. Only the presence of the annotation is
checked.

## Labels

| Label | Default | Purpose |
|---|---|---|
| `kube-descheduler` | `'false'` | Enable the policy |
| `kube-descheduler-subscription-name`, `-channel`, `-source`, `-source-namespace` | operator defaults | Standard operator labels |
| `kube-descheduler-version` | `clusterkubedescheduleroperator.v5.4.3` | CSV pin. Remove it to follow the channel |
| `kube-descheduler-psi` | `'false'` | Add the `psi=1` kernel argument. Restarts nodes |
| `kube-descheduler-psi-role` | `worker` | Machine config pool for the kernel argument |

## Verify

```bash
oc get policy -A | grep kube-descheduler
oc get kubedescheduler cluster -n openshift-kube-descheduler-operator \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.message}{"\n"}{end}'
oc get pods -n openshift-kube-descheduler-operator
```

## Red Hat documentation

- [Descheduler](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/nodes/_descheduler)
- [Enabling descheduler evictions on virtual machines](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/virtualization/virt-enabling-descheduler-evictions)
