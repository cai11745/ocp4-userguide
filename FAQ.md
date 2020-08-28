### 1. web不能查看pod日志
web页面报错
An error occurred while retrieving the requested logs.

命令报错
[root@bastion ~]# oc logs openjdk-app-6-build
Error from server: Get https://192.168.2.23:10250/containerLogs/ademo/openjdk-app-6-build/sti-build: remote error: tls: internal error

解决：
[root@bastion ~]# oc  get csr --all-namespaces
[root@bastion ~]# oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve

### 2. s2i build 报错
简述: 没有内部仓库，创建内部镜像仓库

现象：
Generated from build-controller
82 times in the last hour
Error starting build: an image stream cannot be used as build output because the integrated container image registry is not configured
解决：先部署内部仓库 docker-registry，见手册之镜像仓库

### 3. podman push 失败，仓库是 dockerhub的 registry 镜像
[root@bastion deploy]# podman push registry.example.com:5000/external_storage/nfs-client-provisioner:latest
Getting image source signatures
Copying blob a3ed95caeb02 skipped: already exists
Copying blob a3ed95caeb02 skipped: already exists
Copying blob a17ae64bae4f skipped: already exists
Copying blob 8dfad2055603 skipped: already exists
Copying blob bd01fa00617b [--------------------------------------] 0.0b / 0.0b
Writing manifest to image destination
Error: Error copying image to the remote destination: Error writing manifest: Error uploading manifest latest to registry.example.com:5000/external_storage/nfs-client-provisioner: received unexpected HTTP status: 500 Internal Server Error
解决：
在仓库的日志看到这个，网上查询是要版本换成2.6，新的2.7 不支持
time="2020-06-29T13:29:37.901639587Z" level=error msg="response completed with error" auth.user.name=root err.code=unknown err.detail="manifest schema v1 unsupported" err.message="unknown error"

重新推送镜像成功
Writing manifest to image destination
DEBU[0000] PUT https://registry.example.com:5000/v2/external_storage/nfs-client-provisioner/manifests/latest
Storing signatures
DEBU[0001] Successfully pushed docker://registry.example.com:5000/external_storage/nfs-client-provisioner:latest with digest sha256:b61d360618259b48e8a69fd32c44ef7b10cd0378e3779518821e87e0c84b9c73

### 4. podman build 报错 panic: runtime error: invalid memory address or nil pointer dereference
现象：
[root@bastion es-dockerfile]# podman build -t registry.example.com:5000/openshift4/ose-logging-elasticsearch6:v1 .
STEP 1: FROM registry.example.com:5000/openshift4/ose-logging-elasticsearch6@sha256:88e6b0b164f8f798032b82f97c23c77756eead60a79764d2a15a449f8423c1b6
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x58 pc=0x5633eec25f18]

解决：
升级podman 版本，当前 1.6.4 ，升级到当前最新版本 2.0.5
[root@bastion es-dockerfile]# podman version
Version:            1.6.4
RemoteAPI Version:  1
Go Version:         go1.12.12
OS/Arch:            linux/amd64

[root@bastion es-dockerfile]# podman version
Version:      2.0.5
API Version:  1
Go Version:   go1.13.14
Built:        Thu Jan  1 08:00:00 1970
OS/Arch:      linux/amd64

### 5. oc command 返回 Unable to connect to the server: x509: certificate has expired or is not yet valid
现象：
[root@bastion ~]# oc get node
Unable to connect to the server: x509: certificate has expired or is not yet valid

但是web console 是正常的，可以通过页面登录，集群没有问题。

解决：
清理缓存，重新 oc login 之后正常了
```bash
cd /root/.kube/
rm -rf http-cache/*
rm -rf cache/*

oc login https://api.ocp4.example.com:6443
Use insecure connections? (y/n): y

 oc get node
NAME                        STATUS   ROLES           AGE   VERSION
master-0.ocp4.example.com   Ready    master,worker   45d   v1.17.1+912792b
master-1.ocp4.example.com   Ready    master,worker   45d   v1.17.1+912792b
master-2.ocp4.example.com   Ready    master,worker   45d   v1.17.1+912792b
```
