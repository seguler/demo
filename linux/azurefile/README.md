# Dynamic Provisioning for azure file in Linux (support from v1.7.0)
## 1. create a storage class for azure file
There are two options for creating azure file storage class
#### Option#1: find a suitable storage account that matches ```skuName``` in same resource group when provisioning azure file
```
kubectl create -f https://raw.githubusercontent.com/andyzhangx/Demo/master/pv/storageclass-azurefile.yaml
```

#### Option#2: use existing storage account when provisioning azure file
download `storageclass-azurefile-account.yaml` file and modify `storageAccount` values
```
wget https://raw.githubusercontent.com/andyzhangx/Demo/master/pv/storageclass-azurefile-account.yaml
vi storageclass-azurefile-account.yaml
kubectl create -f storageclass-azurefile-account.yaml
```

## 2. create a pvc for azure file first
```kubectl create -f https://raw.githubusercontent.com/andyzhangx/Demo/master/pv/pvc-azurefile.yaml```

#### make sure pvc is created successfully
```watch kubectl describe pvc pvc-azurefile```

## 3. create a pod with azure file pvc
```kubectl create -f https://raw.githubusercontent.com/andyzhangx/Demo/master/linux/azurefile/nginx-pod-azurefile.yaml```

#### watch the status of pod until its Status changed from `Pending` to `Running`
```watch kubectl describe po nginx-azurefile```

## 4. enter the pod container to do validation
```kubectl exec -it nginx-azurefile -- bash```

```
root@nginx-azurefile:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          30G  3.6G   26G  13% /
tmpfs           6.9G     0  6.9G   0% /dev
tmpfs           6.9G     0  6.9G   0% /sys/fs/cgroup
/dev/sda1        30G  3.6G   26G  13% /etc/hosts
/dev/sdc        4.8G   12M  4.6G   1% /mnt/blobfile
shm              64M     0   64M   0% /dev/shm
tmpfs           6.9G   12K  6.9G   1% /run/secrets/kubernetes.io/serviceaccount
```
### Known issues of azure file dynamic provision
1. There is a [bug](https://github.com/kubernetes/kubernetes/pull/48326) of azure file dynamic provision in [v1.7.0, v1.7.10] (fixed in v1.7.11 or above, v1.8.0): cluster name length must be less than 16 characters, otherwise following error will be received when creating dynamic privisioning azure file pvc:
```
persistentvolume-controller    Warning    ProvisioningFailed Failed to provision volume with StorageClass "azurefile": failed to find a matching storage account
```

2. To specify a storage account in azure file dynamic provision, you should make sure the specified storage account is in the same resource group as your k8s cluster, if you are using AKS, the specified storage account should be in `shadow resource group`(naming as `MC_+{RESOUCE-GROUP-NAME}+{CLUSTER-NAME}+{REGION}`) which contains all resources of your aks cluster. 

# Static Provisioning for azure file in Linux (support from v1.5.0)
kubernetes v1.5, v1.6 does not support dynamic provisioning for azure file, only static provisioning is supported for azure file which means a storage account should be created before using azure file mount feature.

## 1. create a secret for azure file
1) create an azure file share in Azure storage account in the same resource group with k8s cluster, get connection info of that azure file
2) create a `azure-secrect.yaml` file that contains base64 encoded Azure Storage account name and key.

download `azure-secrect.yaml` file and modify `azurestorageaccountname`, `azurestorageaccountkey` values
```
wget https://raw.githubusercontent.com/andyzhangx/Demo/master/pv/azure-secrect.yaml
vi azure-secrect.yaml
```

In the secret file, base64-encode azurestorageaccountname and azurestorageaccountkey. 
For the base64-encode, you could leverage this site: https://www.base64encode.net/

3. create the secret for azure file
```
kubectl create -f azure-secrect.yaml
```

## 2. create a pod with azure file
```kubectl create -f https://raw.githubusercontent.com/andyzhangx/Demo/master/linux/azurefile/nginx-pod-azurefile-static.yaml```

#### watch the status of pod until its `Status` changed from `Pending` to `Running`
```watch kubectl describe po nginx-azurefile```

## 3. enter the pod container to do validation
```kubectl exec -it nginx-azurefile -- bash```

```
root@nginx-azurefile:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          30G  3.6G   26G  13% /
tmpfs           6.9G     0  6.9G   0% /dev
tmpfs           6.9G     0  6.9G   0% /sys/fs/cgroup
/dev/sda1        30G  3.6G   26G  13% /etc/hosts
/dev/sdc        4.8G   12M  4.6G   1% /mnt/blobfile
shm              64M     0   64M   0% /dev/shm
tmpfs           6.9G   12K  6.9G   1% /run/secrets/kubernetes.io/serviceaccount
root@nginx-azurefile:/# mount | grep cifs
//pvc3329812692002.file.core.windows.net/andy-mgwin1710-dynamic-pvc-7b5346be-d577-11e7-bc95-000d3a041274 on /mnt/blobfile type cifs (rw,relatime,vers=3.0,cache=strict,username=pvc3329812692002,domain=,uid=0,noforceuid,gid=0,noforcegid,addr=52.239.184.8,file_mode=0777,dir_mode=0777,persistenthandles,nounix,serverino,mapposix,rsize=1048576,wsize=1048576,echo_interval=60,actimeo=1)
```

### Other known issues of Azure file feature
1. `Premium` storage type is not supported for azure file currently

2. [Azure file on Sovereign Cloud](https://github.com/kubernetes/kubernetes/pull/48460) is supported from v1.7.11, v1.8.0

3. `fileMode`, `dirMode` value would be different in different versions, in latest master branch, it's `0755` by default, to set a different value, follow this [mount options support of azure file](https://github.com/andyzhangx/Demo/blob/master/linux/azurefile/azurefile-mountoptions.md) (available from v1.8.5). For version v1.8.0-v1.8.4, since [mount options support of azure file](https://github.com/andyzhangx/Demo/blob/master/linux/azurefile/azurefile-mountoptions.md) is not available, as a workaround, you could specify a [securityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) for the container with: `runAsUser: 0`, here is an [example](https://github.com/andyzhangx/Demo/blob/master/linux/azurefile/demo-azurefile-securitycontext.yaml)

| version | `fileMode`, `dirMode` value |
| ---- | ---- |
| v1.6.x, v1.7.x | 0777 |
| v1.8.0-v1.8.5 | 0700 |
| v1.8.6 or above | 0755 |
| v1.9.0 | 0700 |
| v1.9.1 or above | 0755 |


#### Links
[Azure File Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/#azure-file)

[Azure file introduction](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-introduction)

[Azure Files scale targets](https://docs.microsoft.com/en-us/azure/storage/common/storage-scalability-targets#azure-files-scale-targets)

[Persistent volumes with Azure files - dynamic provisioning](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv)

[Using Azure Files with Kubernetes](https://docs.microsoft.com/en-us/azure/aks/azure-files)

