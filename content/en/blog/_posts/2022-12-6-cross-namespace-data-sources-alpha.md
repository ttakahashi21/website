---
layout: blog
title: "Kubernetes v1.26: alpha support for cross namespace data sources"
date: 2022-12-6
slug: cross-namespace-data-sources-alpha-redesigned
---

**Authors:**
Takafumi Takahashi (Hitachi Vantara)

Kubernetes v1.26, released earlier this month, introduced an enable the usage of
cross namespace volume data source to allow you to specify a namespace
in the `dataSourceRef` field of a PersistentVolumeClaim.
Before Kubernetes v1.26, by using `AnyVolumeDataSource` feature,
users can provision volumes from data source in the same namespace.
However, it only works for the data source in the same namespace,
therefore users can't provision a persisten volume claim
in one namespace from a data source in other namespace.
To solve this problem, Kubernetes v1.26 adds a new namespace alpha field
to `dataSourceRef` field in PersistentVolumeClaim API.

## How it works

Once the csi-provisioner finds that a data source is specified with a `dataSourceRef` that
has a non-empty namespace name,
it checks all reference grants within the namespace that's specified by the`.spec.dataSourceRef.namespace`
field of the PersistentVolumeClaim, in order to see if access to the data source is allowed.
If any ReferenceGrant allows access, the csi-provisioner provisions a volume from the data source.

## Trying it out

The following things are required to use cross namespace volume provisioning:

* Enable the `AnyVolumeDataSource` and `CrossNamespaceVolumeDataSource` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) for the kube-apiserver, kube-controller-manager
* Install a CRD for the specific `VolumeSnapShot` controller
* Install the CSI Provisioner controller and enable the `CrossNamespaceVolumeDataSource` feature gate
* Install the CSI driver
* Install a CRD for ReferenceGrants

## Putting it all together

To see how this works, you can install the sample and try it out.
This sample do to create PVC in dev namespace from VolumeSnapshot in prod namespace.
That is a simple example. For real world use, you might want to use a more complex approach.

### Prerequisites

* Kubernetes cluster is being deployed with `AnyVolumeDataSource` and `CrossNamespaceVolumeDataSource` features enabled
* There are two namespaces, dev and prod
* CSI driver is being deployed
* There is an acquired VolumeSnapShot named of "new-snapshot-demo" on prod
* ReferenceGrants CRD is being deployed

### Add referencegrants permission to CSI Provisioner RBAC

Access to referencegrants is only needed when the CSI driver
has the `CrossNamespaceVolumeDataSource` controller capability.
Therefore, external-provisioner adds "get", "list", "watch"
permissions for "referencegrants" on "gateway.networking.k8s.io".

```yaml
  - apiGroups: ["gateway.networking.k8s.io"]
    resources: ["referencegrants"]
    verbs: ["get", "list", "watch"]
```

### The CSI Provisioner controller and enable the CrossNamespaceVolumeDataSource feature gate

Add `--feature-gates=CrossNamespaceVolumeDataSource=true` to csi-provisioner container

```yaml
      - args:
        - -v=5
        - --csi-address=/csi/csi.sock
        - --feature-gates=Topology=true
        - --feature-gates=CrossNamespaceVolumeDataSource=true
        image: csi-provisioner:latest
        imagePullPolicy: IfNotPresent
        name: csi-provisioner
```

### Create a ReferenceGrant. Here's a manifest for an example ReferenceGrant.

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: ReferenceGrant
metadata:
  name: allow-prod-pvc
  namespace: prod
spec:
  from:
  - group: ""
    kind: PersistentVolumeClaim
    namespace: dev
  to:
  - group: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: new-snapshot-demo
```

### Create a PersistentVolumeClaim by using cross namespace data source

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
  namespace: dev
spec:
  storageClassName: csi-hostpath-sc
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  dataSourceRef:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: new-snapshot-demo
    namespace: prod
  volumeMode: Filesystem
```

## How can I learn more?

The enhancement proposal,
[Provision volumes from cross-namespace snapshots](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3294-provision-volumes-from-cross-namespace-snapshots), includes lots of detail about the history and technical implementation of this feature.

Please get involved by joining the Kubernetes SIG Storage to help us enhance this
feature. There are a lot of good ideas already and we'd be thrilled to have more!

## Acknowledgment

It takes a wonderful group to make wonderful software.
Thanks are due to everyone who has contributed to the CrossNamespaceVolumeDataSouce feature,
especially (in alphabetical order) Tim Hockin, Masaki Kimura, Michelle Au and Xing Yang.
It’s been a joy to work with y’all on this.
