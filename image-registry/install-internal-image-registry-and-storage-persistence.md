## 创建内部image-registry并使用持久化存储
按照官方说明，如果安装的时候没有提供共享存储，为了保证 openshift-installer 的安装完成，OpenShift Image Registry Operator 将自身配置为 Removed。 就是说没有安装内部image-registry。  
所以我们需要手动来配置一下。  

操作基于 openshift4.7  
官方文档
https://access.redhat.com/documentation/zh-cn/openshift_container_platform/4.7/html/registry/configuring-registry-storage-baremetal

### 1. 更改镜像 registry 的管理状态
要启动镜像 registry，需要把 Image Registry Operator 配置的 managementState 从 Removed 改为 Managed。

将 managementState Image Registry Operator 配置从 Removed 改为 Managed。例如：
```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
```

### 2. 存储准备
对于不提供默认存储的平台，Image Registry Operator 最初将不可用。安装后，您必须配置 registry 使用的存储，这样 Registry Operator 才可用。

使用storageclass 提供动态存储，或者提前手动创建一个 100Gi的 RWX 属性的pv，用作给image-registry挂载。

[nfs-provisioner提供storageclass动态存储](../storage/nfs-provisioner提供storageclass动态存储.md)

如果没有持久化存储，使用 emptydir，但是image-registry这个 pod 迁移或重建后数据就丢了，配置方法见下文。 empty 模式下副本数不允许超过1

### 3. 创建内部仓库
确认当前没有registry的pod，operator 那个不是仓库服务
 ` oc get pod -n openshift-image-registry `


修改配置文件，在 Spec 中，找到storage 那行，改成下面新的三行内容 storage，pvc，claim 
```bash
[root@bastion ~]# oc edit configs.imageregistry.operator.openshift.io

apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
 ...
spec:
  ...
  managementState: Managed
  ...
  storage: {}

把   storage: {} 改成
  storage:
    pvc:
      claim:
```
managementState: Managed  表示安装内部仓库  
storage-pvc-claim 表示使用持久化存储，会自动创建pvc

如果使用 emptydir，即不使用持久化存储， storage 参数改成如下
```bash
    storage:
      emptyDir: {}
```

注意 storage 参数必须配置，不然 pod 无法正常启动。

保存的返回结果是提示修改成功
```bash
[root@bastion ~]# oc edit configs.imageregistry.operator.openshift.io
config.imageregistry.operator.openshift.io/cluster edited
```

查看pod 和 pvc，能看到 image-registry pod

```bash
[root@bastion ~]# oc project openshift-image-registry
[root@bastion ~]# oc get pod
NAME                                              READY   STATUS      RESTARTS   AGE
cluster-image-registry-operator-d54df55b8-7xw9l   1/1     Running     0          24h
image-pruner-1617926400-cg2kl                     0/1     Completed   0          166m
image-registry-6959c9cb4f-4v8tm                   1/1     Running     0          63s
node-ca-45hbg                                     1/1     Running     0          24h
node-ca-478nr                                     1/1     Running     0          24h
node-ca-bfxgs                                     1/1     Running     0          24h
node-ca-c4zh8                                     1/1     Running     0          23h
node-ca-r8r99                                     1/1     Running     0          23h

[root@bastion ~]# oc get pvc
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
image-registry-storage   Bound    pvc-b4f1e11e-d038-41cd-a63a-f38c318c6ff5   100Gi      RWX            managed-nfs-storage   77s

# 进入容器查看下挂载是否成功  

[root@bastion ~]# oc rsh image-registry-6959c9cb4f-4v8tm 

sh-4.4$ df -h |grep nfs
192.168.2.29:/nfs/storageclass/openshift-image-registry-image-registry-storage-pvc-b4f1e11e-d038-41cd-a63a-f38c318c6ff5  492G  289G  203G  59% /registry

如果出现这个报错，需要手动 approve Pending 证书
[root@bastion ~]# oc rsh image-registry-6959c9cb4f-4v8tm 
Error from server: error dialing backend: remote error: tls: internal error

手动approve pending状态的证书
oc get csr |grep Pending |awk -F ' ' '{print $1}'| xargs oc adm certificate approve

avaiable 变成了true，只managed但是不配存储，状态是false的
[root@bastion ~]# oc get clusteroperator image-registry
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
image-registry   4.7.5     True        False         False      6m11sm
```