---
layout: blog
title: "Kubernetes 1.26: alpha support for provision volumes from cross-namespace snapshots"
date: 2022-12-6
slug: provision-volumes-from-cross-namespace-snapshots-redesigned
---

**Authors:**
Takafumi Takahashi (Hitachi Vantara)

Kubernetes v1.26, released earlier this month, introduced an enable usage of provision of persisten volume claim from volume snapshot in other namespaces.  
Before Kubernetes v1.25, by using volume snapshots feature, users can provision volumes from snapshots. However, it only works for the `VolumeSnapshot` in the samme namespace, therefore users can't provision a persisten volume claim in one namespace from a `VolumeSnapshot` in other namespace. On the other hand, there are use cases that require to share the `VolumeSnapshot` across namespaces.  
 To solve this problem,Kubernetes v1.26 includes a new API field called `dataSourceRef2`.

## How it works

Once the csi-provisioner finds a VolumeSnapshot is specified with non-empty namespace as dataSourceRef2, it checks all ReferenceGrants in　PersistentVolumeClaim.spec.dataSourceRef2.namespace to see if access to the snapshot is allowed. If it is allowed, the csi-provisioner provisions a volume from the snapshot.  

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
