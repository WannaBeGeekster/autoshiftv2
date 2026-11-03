# oadp AutoShift Policy

## Overview

Installs the OpenShift API for Data Protection (OADP) operator, which runs Velero, in the
`openshift-adp` namespace. OADP backs up and restores Kubernetes objects and persistent volume data
to object storage.

This policy installs the operator only. The `DataProtectionApplication` that deploys Velero, the
backup storage locations, and the backup schedules depend on each environment's object storage and
are configured separately.

## Enable on clusters

Set the labels in a clusterset or cluster values file:

```yaml
hubClusterSets:
  hub:
    labels:
      oadp: 'true'
      oadp-subscription-name: redhat-oadp-operator
      oadp-channel: stable
      oadp-source: redhat-operators
      oadp-source-namespace: openshift-marketplace
      # oadp-version: 'oadp-operator.v1.6.2'   # optional: pin a CSV for controlled upgrades
```

Labels live in values files only; the `cluster-labels` policy applies them to managed clusters.

## Install mode

OADP supports only the OwnNamespace install mode, so the policy creates an OperatorGroup that
targets `openshift-adp` itself.

## Version control

The policy follows AutoShift's operator version control. Without `oadp-version`, OLM follows the
channel. With it, the subscription pins to that CSV and upgrades are manual. To list available
versions:

```bash
oc get packagemanifests redhat-oadp-operator -n openshift-marketplace \
  -o jsonpath='{range .status.channels[*]}{.name}={.currentCSV}{"\n"}{end}'
```

## Verification

```bash
oc get policy -A | grep oadp
oc get csv -n openshift-adp
oc get crd dataprotectionapplications.oadp.openshift.io
```

## Next step

Create a `DataProtectionApplication` in `openshift-adp` that points at your object storage
(for example an S3-compatible bucket), then `Schedule` or `Backup` resources for the namespaces to
protect. See the OADP documentation for the provider-specific settings.
