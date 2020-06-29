
通过openshift 自带的模板 catalog 或者 from git，database， 都会使用到imagestream    
离线环境下 imagestream 关联的镜像在初始部署中未曾下载， 需要在部署完成后手动补充。

### 获取镜像列表
系统自带的imagestream 都在 openshift project下  
查看镜像


### 导入 mirror 配置

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

# 导入
oc create -f redhat-io-mirror.yaml    
```

worker 已经更新. master 未完成，还是updating 状态，应该是因为我的单master环境，没有做到平滑升级。  
```bash
[root@bastion ~]# oc get machineconfigpools.machineconfiguration.openshift.io -A
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-917470929e092e1c19e2f0278eeff9f1   False     True       False      1              0                   0                     0                      14h
worker   rendered-worker-d78abe00042f9f1ff634ea4ba1da07e1   True      False      False      1              1                   1                     0                      14h
```

确认节点已更新配置，节点的仓库配置在 /etc/containers/registries.conf

```bash
[root@bastion ~]# ssh core@192.168.2.23

[core@worker0 ~]$ cat /etc/containers/registries.conf
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

[[registry]]
  prefix = ""
  location = "registry.redhat.io"
  mirror-by-digest-only = true

  [[registry.mirror]]
    location = "registry.example.com:5000"
```

podman pull 测试




## 仓库mirror 与 insec

image.config.openshift.io/cluster
ImageContentSourcePolicy

这些配置通过 Machine Config Operator (MCO) 进行管理生效，所以先确认 machine-config 正常， available 要是true

[root@bastion ~]# oc get clusteroperator machine-config
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
machine-config   4.4.6     False       False         True       22h

https://docs.openshift.com/container-platform/4.4/openshift_images/image-configuration.html


### mirror 


### insec

oc edit image.config.openshift.io/cluster

spec 里加
  registrySources:
    insecureRegistries:
    - registry.ocp4.example.com:5000





### 



https://docs.openshift.com/container-platform/4.3/registry/configuring_registry_storage/configuring-registry-storage-baremetal.html




服务的日志在日常运维与异常排查中非常重要，以及在运维或者异常情况下，我们需要进入容器查看我们的文件或者执行相关命令进行排查。

### 查看服务日志
查看服务下所有实例的日志  
应用中心--服务管理，选择服务名称进入服务详情，在服务详情页面点击更多，会看到"服务日志"选项  
![deployment-log-1.png](../images/application/deployment-log-1.png)

日志内容，可以看出，两个实例的日志都有

![deployment-log-2.png](../images/application/deployment-log-2.png)

### 查看实例日志
查看单个实例的日志，实例日志分两部分，一个是标准输出，即程序启动输出到前台的日志，第二个是日志文件，日志文件需要在服务发布的时候进行配置才会采集。  
日志文件采集配置可以参照此文中存储卷的说明。 [通过镜像发布服务](../application/deploy-from-image.md)

下图两个按钮分别对应查看启动日志和日志文件。 
![pod-logs.png](../images/application/pod-logs.png)

**启动日志**
![pod-logs-start.png](../images/application/pod-logs-start.png)

**日志文件**
比如我们对服务配置了存储卷，类型为日志，采集路径为 /tmp/
![pod-logs-config.png](../images/application/pod-logs-config.png)

我们进入容器终端，手动写入两个 .log 文件，测试采集
![pod-logs-terminal](../images/application/pod-logs-terminal.png)

点击pod 右侧的 "日志" 查看日志文件采集



### 进入容器终端
对于已经处于 running 状态的容器，我们可以通过管理平台的终端功能进入到容器内，查看容器内文件，通过已有命令排查问题。  
进入方式有两种：  
1. 应用中心--服务管理，在pod 名称左侧的向下箭头点开，会展示容器，点击最后面的 "终端"

![terminal-enter-1.png](../images/application/terminal-enter-1.png)

2. 应用中心--实例管理，选择相应实例，在实例详情页，点击右下角 "终端"

![terminal-enter-2.png](../images/application/terminal-enter-2.png)

进入终端后，可以执行命令和查看文件  
需要注意两点：  
1. 容器内部不是完整的linux系统，不是所有命令都有
2. 在终端里修改的文件，在后续重启、服务更新后会丢失，配置文件的修改需要通过配置中心或者镜像文件固化。

![terminal-show.png](../images/application/terminal-show.png)
