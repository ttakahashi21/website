---
layout: blog
title: "Kubernetes 1.26: alpha support for provision volumes from cross-namespace snapshots"
date: 2022-12-9
slug: cross-namespace-volume-provisioning-redesigned
---

**Authors:**
Takafumi Takahashi (Hitachi Vantara)

Kubernetes v1.26, released earlier this month, introduced an enable usage of provision of persisten volume claim from volume snapshot in other namespaces.  
Before Kubernetes v1.25, by using volume snapshots feature, users can provision volumes from snapshots. However, it only works for the `VolumeSnapshot` in the samme namespace, therefore users can't provision a persisten volume claim in one namespace from a `VolumeSnapshot` in other namespace. On the other hand, there are use cases that require to share the `VolumeSnapshot` across namespaces.  
 To solve this problem,Kubernetes v1.26 includes a new API field called `dataSourceRef2`.

## TBD Data sources Ref2

Earlier Kubernetes releases already added a `dataSource` and `dataSourceRef` field into the
[PersistentVolumeClaim](/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) API,
used for cloning volumes and creating volumes from snapshots. You could use the `dataSource` field when

Earlier Kubernetes releases already added a `dataSource` field into the
[PersistentVolumeClaim](/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) API,
used for cloning volumes and creating volumes from snapshots. You could use the `dataSource` field when
creating a new PVC, referencing either an existing PVC or a VolumeSnapshot in the same namespace.
That also modified the normal provisioning process so that instead of yielding an empty volume, the
new PVC contained the same data as either the cloned PVC or the cloned VolumeSnapshot.

Volume populators embrace the same design idea, but extend it to any type of object, as long
as there exists a [custom resource](/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
to define the data source, and a populator controller to implement the logic. Initially,
the `dataSource` field was directly extended to allow arbitrary objects, if the `AnyVolumeDataSource`
feature gate was enabled on a cluster. That change unfortunately caused backwards compatibility
problems, and so the new `dataSourceRef` field was born.

In v1.22 if the `AnyVolumeDataSource` feature gate is enabled, the `dataSourceRef` field is
added, which behaves similarly to the `dataSource` field except that it allows arbitrary
objects to be specified. The API server ensures that the two fields always have the same
contents, and neither of them are mutable. The differences is that at creation time
`dataSource` allows only PVCs or VolumeSnapshots, and ignores all other values, while
`dataSourceRef` allows most types of objects, and in the few cases it doesn't allow an
object (core objects other than PVCs) a validation error occurs.

When this API change graduates to stable, we would deprecate using `dataSource` and recommend
using `dataSourceRef` field for all use cases.
In the v1.22 release, `dataSourceRef` is available (as an alpha feature) specifically for cases
where you want to use for custom volume populators.

## TBD Using populators

Every volume populator must have one or more CRDs that it supports. Administrators may
install the CRD and the populator controller and then PVCs with a `dataSourceRef` specifies
a CR of the type that the populator supports will be handled by the populator controller
instead of the CSI driver directly.

Underneath the covers, the CSI driver is still invoked to create an empty volume, which
the populator controller fills with the appropriate data. The PVC doesn't bind to the PV
until it's fully populated, so it's safe to define a whole application manifest including
pod and PVC specs and the pods won't begin running until everything is ready, just as if
the PVC was a clone of another PVC or VolumeSnapshot.

## TBD How it works

PVCs with data sources are still noticed by the external-provisioner sidecar for the
related storage class (assuming a CSI provisioner is used), but because the sidecar
doesn't understand the data source kind, it doesn't do anything. The populator controller
is also watching for PVCs with data sources of a kind that it understands and when it
sees one, it creates a temporary PVC of the same size, volume mode, storage class,
and even on the same topology (if topology is used) as the original PVC. The populator
controller creates a worker pod that attaches to the volume and writes the necessary
data to it, then detaches from the volume and the populator controller rebinds the PV
from the temporary PVC to the orignal PVC.

## Trying it out

The following things are required to use cross namespace volume provisioning:

* Enable the `CrossNamespaceVolumeDataSource` feature gate
* Install a CRD for the specific `VolumeSnapShot controller`
* Install the `VolumeSnapShot controller` itself
* Install the `External Provisioner controller` itself
* Install the `Container Stroge Interface Driver` itself

## Putting it all together

To see how this works, you can install the sample and try it out.
This sample do to create PVC in ns1 namespace from VolumeSnapshot in default namespace by using csi-hostpath driver.

1. Deploy VolumeSnapshot controller.

   ```terminal
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml


   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
   ```

2. Deploy External Provisioner controller.

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

    ```terminal
    cd /tmp/csi-driver-host-path
    kubectl apply -f examples/csi-storageclass.yaml
    kubectl apply -f examples/csi-pvc.yaml
    kubectl apply -f examples/csi-snapshot-v1beta1.yaml
    ```

6. Ceate a ns1 namespace

    ```terminal
    kubectl create ns ns1
    ```

7. Create a ReferenceGrant

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1alpha2
   kind: ReferenceGrant
   metadata:
     name: bar
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
     storageClassName: csi-hostpath-sc
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
     dataSourceRef2:
       apiGroup: snapshot.storage.k8s.io
       kind: VolumeSnapshot
       name: new-snapshot-demo
       namespace: default
     volumeMode: Filesystem
   ```

Note This is only the simplest example.

## How can I learn more?

The enhancement proposal,
[Provision volumes from cross-namespace snapshots](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3294-provision-volumes-from-cross-namespace-snapshots), includes lots of detail about the history and technical implementation of this feature.

Please get involved by joining the Kubernetes storage SIG to help us enhance this
feature. There are a lot of good ideas already and we'd be thrilled to have more!
