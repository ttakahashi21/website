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
in the `dataSourceRef` field.
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
This sample do to create PVC in ns1 namespace from VolumeSnapshot in default namespace by using csi-hostpath driver.

1. Deploy VolumeSnapshot CRD and controller.

   ```terminal
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml


   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
   ```

2. Deploy CSI Provisioner controller.

   ```terminal
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-provisioner/master/deploy/kubernetes/deployment.yaml 
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-provisioner/master/deploy/kubernetes/rbac.yaml
   ```

3. Deploy csi-hostpath driver.

   ```terminal
   cd /tmp
   git clone --depth=1 https://github.com/kubernetes-csi/csi-driver-host-path.git
   cd csi-driver-host-path
   ./deploy/kubernetes-latest/deploy.sh
   ```

4. Deploy ReferenceGrants CRD
  
   ```terminal
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/main/config/ crd/experimental/gateway.networking.k8s.io_referencegrants.yaml
   ```

5. Create StorageClass, PVC, and VolumeSnapshot by the examples in the csi-hostpath repo
In csi-host-path directory(/tmp/csi-driver-host-path):

    ```terminal
    kubectl apply -f examples/csi-storageclass.yaml
    kubectl apply -f examples/csi-pvc.yaml
    kubectl apply -f examples/csi-snapshot-v1.yaml
    ```

6. Ceate a new namespace named `ns1`

    ```terminal
    kubectl create ns ns1
    ```

7. Create a ReferenceGrant. Here's a manifest for an example ReferenceGrant.

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1beta1
   kind: ReferenceGrant
   metadata:
     name: allow-ns1-pvc
     namespace: default
   spec:
     from:
     - group: ""
       kind: PersistentVolumeClaim
       namespace: ns1
     to:
     - group: snapshot.storage.k8s.io
       kind: VolumeSnapshot
       name: new-snapshot-demo
   ```

8. Create a PersistentVolumeClaim

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: foo-pvc
     namespace: ns1
   spec:
     storageClassName: example
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
     dataSourceRef:
       apiGroup: snapshot.storage.k8s.io
       kind: VolumeSnapshot
       name: new-snapshot-demo
       namespace: default
     volumeMode: Filesystem
   ```

That is a simple example. For real world use, you might want to use a more complex approach.

## How can I learn more?

The enhancement proposal,
[Provision volumes from cross-namespace snapshots](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3294-provision-volumes-from-cross-namespace-snapshots), includes lots of detail about the history and technical implementation of this feature.

Please get involved by joining the Kubernetes SIG Storage to help us enhance this
feature. There are a lot of good ideas already and we'd be thrilled to have more!
