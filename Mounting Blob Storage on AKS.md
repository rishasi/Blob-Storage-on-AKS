# Mounting Blob Storage on AKS



## Theory

Azure Blob can be mounted onto Pods running inside an AKS Cluster. There are mainly 2 drivers which are used for mounting Blobs onto Pods:

1. **Azure Blob Storage <u>CSI</u> Driver**: https://github.com/kubernetes-sigs/blob-csi-driver

2. **Blobfuse <u>FlexVolume</u> Driver** (<b> *Deprecated* </b>): https://github.com/Azure/kubernetes-volume-drivers/tree/master/flexvolume/blobfuse

   > Both the above drivers are out-of-tree volume plugins. However, there are certain limitations with the FlexVolume driver, and the the Kubernetes Storage SIG recommends using the CSI Volume Driver. More information about these can be found here: https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md



![image-20210413131705981](docs/image/image-20210413131705981.png)



**What are the limitations of FlexVolume?**

- FlexVolume requires root access on host machine to install FlexVolume driver files.
- FlexVolume drivers assume all volume mount dependencies, e.g. mount and filesystem tools, are available on the host OS. Installing these dependencies also require root access.

You can continue to use FlexVolume without worry of deprecation, but note that additional features (like topology, snapshots, etc.) will only be added to CSI.



In this document, we will primarily be talking about Blob Storage CSI Driver.

Blob CSI Driver allows Kubernetes to access Azure Storage through one of the following ways:

1. azure-storage-fuse: https://github.com/Azure/azure-storage-fuse
2. NFSv3 (still in preview): https://docs.microsoft.com/en-us/azure/storage/blobs/network-file-system-protocol-support

---



## Version Compatibility

https://github.com/kubernetes-sigs/blob-csi-driver#container-images--kubernetes-compatibility

### Container Images & Kubernetes Compatibility (as of writing this doc: Apr' 2021):

| driver version | Image                                      | supported k8s version | built-in blobfuse version |
| -------------- | ------------------------------------------ | --------------------- | ------------------------- |
| master branch  | mcr.microsoft.com/k8s/csi/blob-csi:latest  | 1.16+                 | 1.3.6                     |
| v1.0.0         | mcr.microsoft.com/k8s/csi/blob-csi:v1.0.0  | 1.16+                 | 1.3.6                     |
| v0.11.0        | mcr.microsoft.com/k8s/csi/blob-csi:v0.11.0 | 1.15+                 | 1.3.6                     |
| v0.10.0        | mcr.microsoft.com/k8s/csi/blob-csi:v0.10.0 | 1.15+                 | 1.3.5                     |



There are some important topics for Parameters, Pre-requisites, etc, for the CSI Driver, which can be looked upon in the following link: https://github.com/kubernetes-sigs/blob-csi-driver#driver-parameters

---



## Installation via *kubectl*:

### Installing an older version (0.10.0)

https://github.com/kubernetes-sigs/blob-csi-driver/blob/master/docs/install-csi-driver-v0.10.0.md

```bash
# rishabh@Azure:~$ curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/blob-csi-driver/v0.10.0/deploy/install-driver.sh | bash -s v0.10.0 --
Installing Azure Blob Storage CSI driver, version: v0.10.0 ...
serviceaccount/csi-blob-controller-sa created
clusterrole.rbac.authorization.k8s.io/blob-external-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/blob-csi-provisioner-binding created
clusterrole.rbac.authorization.k8s.io/blob-external-attacher-role created
clusterrole.rbac.authorization.k8s.io/blob-external-snapshotter-role created
clusterrolebinding.rbac.authorization.k8s.io/blob-csi-snapshotter-binding created
clusterrole.rbac.authorization.k8s.io/csi-blob-controller-secret-role created
clusterrolebinding.rbac.authorization.k8s.io/csi-blob-controller-secret-binding created
serviceaccount/csi-blob-node-sa created
clusterrole.rbac.authorization.k8s.io/csi-blob-node-secret-role created
clusterrolebinding.rbac.authorization.k8s.io/csi-blob-node-secret-binding created
Warning: storage.k8s.io/v1beta1 CSIDriver is deprecated in v1.19+, unavailable in v1.22+; use storage.k8s.io/v1 CSIDriver
csidriver.storage.k8s.io/blob.csi.azure.com created
deployment.apps/csi-blob-controller created
daemonset.apps/csi-blob-node created
Azure Blob Storage CSI driver installed successfully.
```

Have a look at the resources being created:

- ClusterRole and Bindings
- Service Accounts
- CSI Driver Storage resouce
- Deployments: blob-controller
- DaemonSet: blob-node

```bash
# rishabh@Azure:~$ kubectl get all -n kube-system | grep csi
NAME                                       READY   STATUS    RESTARTS   AGE
pod/csi-blob-controller-56887dc558-9zj7q   3/3     Running   0          3m31s
pod/csi-blob-controller-56887dc558-txhbr   3/3     Running   0          3m31s
pod/csi-blob-node-cll2x                    3/3     Running   0          3m28s
pod/csi-blob-node-cxh2b                    3/3     Running   0          3m28s
pod/csi-blob-node-z4c8q                    3/3     Running   0          3m28s
pod/csi-blob-node-z8kwz                    3/3     Running   0          3m28s

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset.apps/csi-blob-node              4         4         4       4            4           kubernetes.io/os=linux        3m29s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/csi-blob-controller   2/2     2            2           3m33s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/csi-blob-controller-56887dc558   2         2         2       3m33s
```

### Uninstalling an older version

```bash
# rishabh@Azure:~$ curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/blob-csi-driver/v0.10.0/deploy/uninstall-driver.sh | bash -s v0.10.0 --
Uninstalling Azure Blob Storage CSI driver, version: v0.10.0 ...
deployment.apps "csi-blob-controller" deleted
daemonset.apps "csi-blob-node" deleted
Warning: storage.k8s.io/v1beta1 CSIDriver is deprecated in v1.19+, unavailable in v1.22+; use storage.k8s.io/v1 CSIDriver
csidriver.storage.k8s.io "blob.csi.azure.com" deleted
serviceaccount "csi-blob-controller-sa" deleted
clusterrole.rbac.authorization.k8s.io "blob-external-provisioner-role" deleted
clusterrolebinding.rbac.authorization.k8s.io "blob-csi-provisioner-binding" deleted
clusterrole.rbac.authorization.k8s.io "blob-external-attacher-role" deleted
clusterrole.rbac.authorization.k8s.io "blob-external-snapshotter-role" deleted
clusterrolebinding.rbac.authorization.k8s.io "blob-csi-snapshotter-binding" deleted
clusterrole.rbac.authorization.k8s.io "csi-blob-controller-secret-role" deleted
clusterrolebinding.rbac.authorization.k8s.io "csi-blob-controller-secret-binding" deleted
serviceaccount "csi-blob-node-sa" deleted
clusterrole.rbac.authorization.k8s.io "csi-blob-node-secret-role" deleted
clusterrolebinding.rbac.authorization.k8s.io "csi-blob-node-secret-binding" deleted
Uninstalled Azure Blob Storage CSI driver successfully.

rishabh@Azure:~$ kubectl get all -n kube-system | grep csi
rishabh@Azure:~$

## No resource left.
```



If there are resources remaining in the cluster, which have not been removed properly, the individual resources from the above list can be removed manually using ```kubectl delete``` command.

### Installing latest version (master)

https://github.com/kubernetes-sigs/blob-csi-driver/blob/master/docs/install-csi-driver-master.md

```bash
# rishabh@Azure:~$ curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/blob-csi-driver/master/deploy/install-driver.sh | bash -s master --
Installing Azure Blob Storage CSI driver, version: master ...
serviceaccount/csi-blob-controller-sa created
clusterrole.rbac.authorization.k8s.io/blob-external-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/blob-csi-provisioner-binding created
clusterrole.rbac.authorization.k8s.io/blob-external-attacher-role created
clusterrole.rbac.authorization.k8s.io/blob-external-snapshotter-role created
clusterrolebinding.rbac.authorization.k8s.io/blob-csi-snapshotter-binding created
clusterrole.rbac.authorization.k8s.io/csi-blob-controller-secret-role created
clusterrolebinding.rbac.authorization.k8s.io/csi-blob-controller-secret-binding created
serviceaccount/csi-blob-node-sa created
clusterrole.rbac.authorization.k8s.io/csi-blob-node-secret-role created
clusterrolebinding.rbac.authorization.k8s.io/csi-blob-node-secret-binding created
Warning: storage.k8s.io/v1beta1 CSIDriver is deprecated in v1.19+, unavailable in v1.22+; use storage.k8s.io/v1 CSIDriver
csidriver.storage.k8s.io/blob.csi.azure.com created
deployment.apps/csi-blob-controller created
daemonset.apps/csi-blob-node created
Azure Blob Storage CSI driver installed successfully.


# rishabh@Azure:~$ kubectl get all -n kube-system | grep csi
pod/csi-blob-controller-ddb5567d4-7hglx    4/4     Running   0          42s
pod/csi-blob-controller-ddb5567d4-pwr6p    4/4     Running   0          42s
pod/csi-blob-node-5snbk                    3/3     Running   0          39s
pod/csi-blob-node-7rvjb                    3/3     Running   0          39s
pod/csi-blob-node-8kgnx                    3/3     Running   0          39s
pod/csi-blob-node-pp46r                    3/3     Running   0          39s

daemonset.apps/csi-blob-node              4         4         4       4            4           kubernetes.io/os=linux        39s

deployment.apps/csi-blob-controller   2/2     2            2           43s

replicaset.apps/csi-blob-controller-ddb5567d4    2         2         2       43s
```



#### To check the version installed

check the ```blob``` container's image in the csi-blob-controller Pod/Deployment:

```bash
# rishabh@Azure:~$ kubectl describe deploy -n kube-system csi-blob-controller
Name:                   csi-blob-controller
Namespace:              kube-system
...
...
   blob:                                                     ## Out of 4 containers in this Pod, check blob.
    Image:       mcr.microsoft.com/k8s/csi/blob-csi:latest   ## This is the version: latest
    Ports:       29632/TCP, 29634/TCP
...
...
```

---



## Usage

