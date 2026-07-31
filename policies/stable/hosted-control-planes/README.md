# Hosted Control Planes (HyperShift)

This AutoShift policy enables **Hosted Control Planes (HyperShift)** on the hub so it can host
OpenShift control planes — for example, KubeVirt-based hosted clusters whose worker nodes run as
VMs on OpenShift Virtualization.

Unlike most AutoShift policies, HyperShift is **not** installed as an OLM operator. It is enabled
by turning on two components of the **MultiClusterEngine (MCE)** that ACM creates: `hypershift`
and `hypershift-local-hosting`. This policy reads the live MCE and flips only those two components
to `enabled`, preserving every other component (a plain `musthave` on the component list would risk
duplicate entries).

## Enabling

Set the toggle in your clusterset values file (e.g. `autoshift/values/clustersets/hub.yaml`):

```yaml
hcp: 'true'
```

The policy is **hub-targeted** — the MCE lives on the hub — and gated by the `autoshift.io/hcp`
label. It takes effect once ACM's MultiClusterHub has created the MultiClusterEngine; until then
the ConfigurationPolicy is a no-op.

## Prerequisites

- **ACM / MultiClusterHub** installed on the hub (this creates the MultiClusterEngine).
- For **KubeVirt** hosted clusters: enable `virt` (OpenShift Virtualization) on the hosting
  cluster(s), plus a LoadBalancer for the hosted cluster's API/ingress — MetalLB (`metallb`) or an
  external load balancer fronting NodePorts.

## What it does

Delivers a single ConfigurationPolicy (`enable-hypershift`) that ensures the MCE's
`spec.overrides.components` has:

- `hypershift` → `enabled: true`
- `hypershift-local-hosting` → `enabled: true`

Enabling `hypershift-local-hosting` also installs the `hypershift-addon` on `local-cluster`, so the
hub can host control planes locally. Creating actual `HostedCluster` / `NodePool` resources is a
separate, per-use step (a follow-on policy can add a sample).

## Hosting on managed (spoke) clusters

To let a **managed cluster** host control planes as well (not just the hub), label it:

```yaml
hcp-hosting: 'true'
```

The `policy-hcp-hypershift-addon` policy iterates the ManagedClusters and creates a
`ManagedClusterAddOn/hypershift-addon` (in `open-cluster-management-agent-addon`) in the namespace of
every cluster labelled `autoshift.io/hcp-hosting: 'true'`, which installs the HyperShift Operator on
that cluster. This policy depends on `policy-hcp-enable-hypershift` (the `hypershift` MCE component
must be on first).

## Sample HostedCluster (optional)

An illustrative KubeVirt `HostedCluster` + `NodePool` (named `sample`, in the `clusters` namespace)
can be created as a starting point. It is **off by default** and gated by its own label:

```yaml
hcp-sample: 'true'
```

Tunables (labels, with defaults):

| Label | Default | Purpose |
|-------|---------|---------|
| `hcp-sample-base-domain` | `apps.example.com` | Hosted cluster base domain |
| `hcp-sample-pull-secret` | `sample-pull-secret` | Pull-secret **Secret name** (create it in `clusters`) |
| `hcp-sample-ssh-key` | `sample-ssh-key` | SSH-key **Secret name** (create it in `clusters`) |
| `hcp-sample-storage-class` | `ocs-storagecluster-ceph-rbd` | StorageClass for etcd + node root volumes |
| `hcp-sample-api-strategy` | `NodePort` | API server publishing: `NodePort` (no LB) / `LoadBalancer` (needs MetalLB/an LB) / `Route` |
| `hcp-sample-release-image` | `…ocp-release:4.18.0-multi` | OpenShift release image |

Secrets are referenced **by name** only — create the pull-secret and SSH-key Secrets in the
`clusters` namespace on the hosting cluster before enabling the sample.
