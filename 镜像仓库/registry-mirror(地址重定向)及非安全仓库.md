## registry-mirror 地址重定向
如果我们的集群是通过离线环境部署的，在部署时的配置文件 install-config.yaml 中，有一个参数 imageContentSources 是把quay.io 的镜像重定向到了我们的私有仓库 registry.example.com  

```bash
cat /opt/install/install-config.yaml.0629
...
imageContentSources:
- mirrors:
  - registry.example.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.example.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
...
```

这两个参数在集群部署后，存在了这里  
```bash
[root@bastion ~]# oc get ImageContentSourcePolicy
NAME             AGE
image-policy-0   4d3h
image-policy-1   4d3h
```

在每个master 或 worker 节点，体现在配置文件  /etc/containers/registries.conf  

```bash
[core@master-2 ~]$ cat  /etc/containers/registries.conf
unqualified-search-registries = ["registry.access.redhat.com", "docker.io"]

[[registry]]
  prefix = ""
  location = "quay.io/openshift-release-dev/ocp-release"
  mirror-by-digest-only = true

  [[registry.mirror]]
    location = "registry.example.com:5000/ocp4/openshift4"

[[registry]]
  prefix = ""
  location = "quay.io/openshift-release-dev/ocp-v4.0-art-dev"
  mirror-by-digest-only = true

  [[registry.mirror]]
    location = "registry.example.com:5000/ocp4/openshift4"
```

### 1. registry-mirror 配置方式
registry-mirror 由上可知是通过 ImageContentSourcePolicy 这种资源方式进行管理的

这个资源对象通过 Machine Config Operator (MCO) 进行管理生效，所以先确认 machine-config 正常， available 要是true

[root@bastion ~]# oc get clusteroperator machine-config
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
machine-config   4.4.6     False       False         True       22h

https://docs.openshift.com/container-platform/4.4/openshift_images/image-configuration.html

### 2. 新增 registry-mirror 
我们新增一个 ImageContentSourcePolicy 把 registry.redhat.io 重定向到私有仓库 registry.example.com:5000  

```bash
[root@bastion ~]# cat redhat-io-mirror.yaml
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: redhat-io
spec:
  repositoryDigestMirrors:
  - mirrors:
    - registry.example.com:5000
    source: registry.redhat.io

# 导入配置
[root@bastion ~]# oc apply -f redhat-io-mirror.yaml
```

此时，node 节点会依次变成不可调度，并更新配置，当所有节点状态都恢复之后，说明配置更新完成  
```bash
# 先是 master2 被设置为禁用调度
[root@bastion ~]# oc get node
NAME                        STATUS                     ROLES           AGE    VERSION
master-0.ocp4.example.com   Ready                      master,worker   4d4h   v1.17.1+912792b
master-1.ocp4.example.com   Ready                      master,worker   4d4h   v1.17.1+912792b
master-2.ocp4.example.com   Ready,SchedulingDisabled   master,worker   4d3h   v1.17.1+912792b

# 过一会切换到了 master0 禁用调度
[root@bastion ~]# oc get node
NAME                        STATUS                     ROLES           AGE    VERSION
master-0.ocp4.example.com   Ready,SchedulingDisabled   master,worker   4d4h   v1.17.1+912792b
master-1.ocp4.example.com   Ready                      master,worker   4d4h   v1.17.1+912792b
master-2.ocp4.example.com   Ready                      master,worker   4d4h   v1.17.1+912792b

```

在配置更新过程中，machine-config 的状态 AVAILABLE 是false， DEGRADED 是true

```bash
[root@bastion ~]# oc get clusteroperator machine-config
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
machine-config   4.4.9     False       False         True       33m
```

所有节点都更新完成后，状态都恢复到 ready，且不会禁用调度。 machine-config 的状态也恢复到了 AVAILABLE True

```bash
[root@bastion ~]# oc get node
NAME                        STATUS   ROLES           AGE    VERSION
master-0.ocp4.example.com   Ready    master,worker   4d5h   v1.17.1+912792b
master-1.ocp4.example.com   Ready    master,worker   4d5h   v1.17.1+912792b
master-2.ocp4.example.com   Ready    master,worker   4d5h   v1.17.1+912792b
[root@bastion ~]# oc get clusteroperator machine-config
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
machine-config   4.4.9     True        False         False      12s
```

查看所有节点配置文件，会看到我们新增的配置
```bash
[core@master-2 ~]$ cat  /etc/containers/registries.conf
...

[[registry]]
  prefix = ""
  location = "registry.redhat.io/rhscl"
  mirror-by-digest-only = true

  [[registry.mirror]]
    location = "registry.example.com:5000/rhscl"
```

### 3. 测试mirror 效果




### 4. insec

oc edit image.config.openshift.io/cluster

spec 里加
  registrySources:
    insecureRegistries:
    - registry.ocp4.example.com:5000



