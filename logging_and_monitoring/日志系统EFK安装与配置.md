## 日志系统EFk安装与配置

本篇介绍 openshift4 日志系统EFK的安装与配置。 openshift4 默认安装没有自带日志系统，需要我们手动通过 operator 安装。  

### EFK 简介
集群日志系统基于 Elasticsearch，Fluentd和Kibana（EFK）。收集器 Fluentd 部署到 OpenShift 集群中的每个节点。它收集所有节点和容器日志，并将它们写入 Elasticsearch（ES）。Kibana 是集中式的Web UI，用户和管理员可以在其中使用汇总的数据创建丰富的可视化效果和仪表板，数据来源于ES。

主要有 5 个不同类型的组件： 
logStore（存储） - 存储日志的位置。当前的实现是 Elasticsearch。
collection（收集） - 此组件从节点收集日志，将日志格式化并存储到 logStore 中。当前的实现是 Fluentd。Fluentd 将 systemd journal 和 /var/log/containers/ 的所有日志都传输到 Elasticsearch。
visualization（可视化） - 此 UI 组件用于查看日志、图形和图表等。当前的实现是 Kibana。
curation（内容管理） - 此组件按日志时间进行筛检，主要用于管理 es 的索引。当前的实现是 Curator。
event routing - 用来把 OpenShift Container Platform 的事件转发到集群日志记录的组件。当前的实现是 Event Router。 （下文不涉及此内容）

如果集群可以联网，可以直接在 web console， operatorhub 页面部署，需要用到 Elasticsearch Operator 和 Cluster Logging Operator。  
离线环境，需先准备离线的 operatorhub 环境，可参照文章 [离线部署operatorhub并批量导入operator](../application_store/离线部署operatorhub并批量导入operator.md)

### 安装 Elasticsearch Operator
在 web console 页面，选择 Operators - OperatorHub，找到 Elasticsearch Operator 点击 Install  
- Installation Mode 选默认，All namespace  
- Installed Namepspace，选 openshift-operators-redhat . 必须是这个 
- Update Channel和 Approval Strategy: 默认 
- 勾选 “Enable operator recommended cluster monitoring on this namespace”
其他默认，点击 Subscribe 进行安装  

校验：  
1. 在菜单 Operators - Installed Operators 中，确保 Elasticsearch Operator 的 Status 是 Succeeded

2. 通过页面 Workloads - pod 或者后台命令，查看 openshift-operators-redhat 这个 project 下会有一个elasticsearch-operator pod，若没有说明有异常  
[root@bastion ~]# oc get pod -n openshift-operators-redhat
NAME                                      READY   STATUS    RESTARTS   AGE
elasticsearch-operator-77dfc94df7-mxpp2   1/1     Running   0          2m38s

若出现异常想重置此步骤，可以先在 Installed Operators 里面，把operator 卸载了，再把 openshift-operators-redhat 这个 project 删掉，比较彻底。
我遇到过 elasticsearch-operator 这个pod 出不来，把 project 删了，重做了一遍就好了。  

### 安装 Cluster Logging Operator
在 web console 页面，选择 Operators - OperatorHub，找到 Cluster Logging 点击 Install  
- Installation Mode 选默认，A specific namespace on the cluster，只在指定namespace 生效  
- Installed Namepspace，选 Operator recommended namespace - openshift-logging  
- Update Channel和 Approval Strategy: 默认 
- 勾选 “Enable operator recommended cluster monitoring on this namespace”
其他默认，点击 Subscribe 进行安装  

校验：  
1. 在菜单 Operators - Installed Operators 中，确保 Cluster Logging 的 Status 是 Succeeded
（如果状态异常，可以在 workloads - pod 里看到一个 pod 名为 cluster-logging-operator-xxx ，通过 pod 的events 或者 logs 进行排查。）  

2. 通过页面 Workloads - pod 或者后台命令，查看 openshift-logging 这个 project 下会有一个cluster-logging-operator pod，若没有说明有异常  

```bash
[root@bastion ~]# oc get pod -n openshift-logging
NAME                                       READY   STATUS    RESTARTS   AGE
cluster-logging-operator-dc666d47c-dqlzh   1/1     Running   0          2m17s
```

### 创建 cluster logging instance
进入 Administration - Custom Resource Definitions，进入 ClusterLogging  
在 Instances 页面选择 Create ClusterLogging,导入以下 yaml 文件，相关参数见说明
(也可以在 Installed Operator，Cluster Logging 的详情页可导入 yaml)

此文件可以定义 EFK 各个组件的参数及资源限制、存储等  
```bash
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance" 
  namespace: "openshift-logging"
spec:
  managementState: "Managed"  
  logStore:
    type: "elasticsearch"  
    retentionPolicy: 
      application:
        maxAge: 7d
      infra:
        maxAge: 7d
      audit:
        maxAge: 7d
    elasticsearch:
      nodeCount: 3 
      resources:
        limits:
          memory: 4Gi
        requests:
          cpu: 200m
          memory: 2Gi
      storage: {}    
      redundancyPolicy: "SingleRedundancy"
  visualization:
    type: "kibana"  
    kibana:
      replicas: 1
  curation:
    type: "curator"
    curator:
      schedule: "30 3 * * *" 
  collection:
    logs:
      type: "fluentd"  
      fluentd: {}
```

都启动完成后 pod 要都是正常的。在 es 集群就绪之前，fluentd 的pod 会一直重启，不用介意。  
```bash
[root@bastion es-dockerfile]# oc get pod -n openshift-logging
NAME                                            READY   STATUS    RESTARTS   AGE
cluster-logging-operator-dc666d47c-dqlzh        1/1     Running   0          8d
elasticsearch-cdm-sf8h63d4-1-77fb57df57-6jtrw   2/2     Running   0          3m21s
elasticsearch-cdm-sf8h63d4-2-84865c4ddf-whcpv   2/2     Running   0          2m20s
elasticsearch-cdm-sf8h63d4-3-797c999f7c-pzrhq   2/2     Running   0          79s
fluentd-872lj                                   1/1     Running   0          3m23s
fluentd-gk55j                                   1/1     Running   0          3m23s
fluentd-pkwl4                                   1/1     Running   0          3m23s
kibana-78f94c4945-4gfw5                         2/2     Running   0          3m19s
```

Curator 是定时任务，通过 oc get cronjob 查看。

**Note:**
如果看不到 elasticsearch pod，是 Elasticsearch Operator 安装有问题，回到上文去确认 openshift-operators-redhat 这个 project 下有没有pod ，及 pod 日志  

如果 elasticsearch pod 生成了，但是状态是 CrashLoopBackOff        
日志中有这个报错：
cp: cannot create regular file '/etc/elasticsearch/elasticsearch.yml': Permission denied
cp: cannot create regular file '/etc/elasticsearch/log4j2.properties': Permission denied
或者
sed: couldn't open temporary file /opt/app-root/src/sgconfig/sed928Aqo: Permission denied
sed: couldn't open temporary file /usr/share/elasticsearch/index_templates/sedBRXL6C: Permission denied
解决方法见最后FAQ，需要重做镜像

### cluster logging instance 参数解读
一共有4个组件，每个组件都可以单独定义资源限额，即 request 和 limit  

metadata:
  name: "instance"  #固定，不要改
  namespace: "openshift-logging"   #固定，不要改

**Elasticsearch 参数**
指定Elasticsearch保留每个日志源的时间长度。输入一个整数和一个时间名称：周（w），小时（h / H），分钟（m）和秒。例如，7d连续七天。早于的日志将maxAge被删除。您必须为每个日志源指定一个保留策略，否则将不会为该源创建Elasticsearch索引。
```bash
    retentionPolicy: 
      application:
        maxAge: 7d
      infra:
        maxAge: 7d
      audit:
        maxAge: 7d
```

es 集群节点数，支持最多 3个  
```bash
      elasticsearch:
        nodeCount: 3

      resources:
        limits:
          memory: 4Gi  #默认值为16G，官方建议生产环境16G。request同
```

存储这样写的话，就是使用 emptyDir 临时卷，重启或者迁移之后数据会丢失。
`        storage: {} `

若要持久化，可以使用 storageclass，避免数据丢失。
```bash
        storage:
          storageClassName: "gp2"
          size: "200G"
```

为分片指定冗余策略。
FullRedundancy: Elasticsearch将每个索引的主要碎片完全复制到每个数据节点。这提供了最高的安全性，但以所需的最大磁盘数量和最差的性能为代价。
MultipleRedundancy: Elasticsearch将每个索引的主要分片完全复制到一半数据节点。这提供了安全性和性能之间的良好折衷。
SingleRedundancy: Elasticsearch为每个索引制作一个主要分片的副本。只要存在至少两个数据节点，日志就始终可用且可恢复。使用5个或更多节点时，性能比MultipleRedundancy好。您不能将此策略应用于单个Elasticsearch节点的部署。
ZeroRedundancy: Elasticsearch不会复制主碎片。如果节点关闭或发生故障，日志可能不可用或丢失。当您更关注性能而不是安全性，或者已实现自己的磁盘/ PVC备份/还原策略时，请使用此模式。

`      redundancyPolicy: "SingleRedundancy" `

其他组件参数，如 kibana 副本数，fluentd 配置 buff，调度到指定节点等，详见官网。

https://docs.openshift.com/container-platform/4.3/logging/config/cluster-logging-configuring.html

### 部署效果验证
EFK 部署成功后，在 pod 详情，logs 页面会多出一个“Show in Kibana”按钮，点击会跳转到 kibana 页面

首次进入 kibana 需要授权，并手动创建索引规则，会看到有 app-xx infra-xx 的索引，如果看不到可以等等，es 那边还没创建好  
![pod-logs](../images/logging_and_monitoring/pod-log-kibana-button.png)

![kibana](../images/logging_and_monitoring/pod-log-on-kibana.png)

也可以在 kibana 页面自己根据规则进行日志检索

查看 ES 集群状态，索引策略，及 client 状态等
```bash
oc get Elasticsearch elasticsearch -n openshift-logging -o yaml

spec:
  indexManagement:
    mappings:
    - aliases:
      - app
      - logs.app
      name: app
      policyRef: app-policy
...
status:
  cluster:
    activePrimaryShards: 186
    activeShards: 372
    initializingShards: 0
    numDataNodes: 3
    numNodes: 3
    pendingTasks: 0
    relocatingShards: 0
    status: green
...
  pods:
    client:
      failed: []
      notReady: []
      ready:
      - elasticsearch-cdm-sf8h63d4-1-77fb57df57-6jtrw
      - elasticsearch-cdm-sf8h63d4-2-84865c4ddf-whcpv
      - elasticsearch-cdm-sf8h63d4-3-797c999f7c-pzrhq
    data:
      failed: []
      notReady: []
      ready:
      - elasticsearch-cdm-sf8h63d4-1-77fb57df57-6jtrw
      - elasticsearch-cdm-sf8h63d4-2-84865c4ddf-whcpv
      - elasticsearch-cdm-sf8h63d4-3-797c999f7c-pzrhq
    master:
      failed: []
      notReady: []
      ready:
      - elasticsearch-cdm-sf8h63d4-1-77fb57df57-6jtrw
      - elasticsearch-cdm-sf8h63d4-2-84865c4ddf-whcpv
      - elasticsearch-cdm-sf8h63d4-3-797c999f7c-pzrhq
  shardAllocationEnabled: all
[root@bastion ~]# 
```

### FAQ  
#### issue1: elasticsearch pod 状态是 CrashLoopBackOff，日志 cp: cannot create regular file '/etc/elasticsearch/elasticsearch.yml': Permission denied
  
问题现象：
es 的 pod 启动异常，查看日志发现是执行 cp/sed 命令时候权限有问题  
报错信息是：  
cp: cannot create regular file '/etc/elasticsearch/elasticsearch.yml': Permission denied
cp: cannot create regular file '/etc/elasticsearch/log4j2.properties': Permission denied
或者  
sed: couldn't open temporary file /opt/app-root/src/sgconfig/sed928Aqo: Permission denied
sed: couldn't open temporary file /usr/share/elasticsearch/index_templates/sedBRXL6C: Permission 

```bash
[root@bastion ~]# oc get pod
NAME                                            READY   STATUS                  RESTARTS   AGE
cluster-logging-operator-dc666d47c-dqlzh        1/1     Running                 0          6d
elasticsearch-cdm-fyglie0e-1-5c86bfd95c-q9zwm   1/2     CrashLoopBackOff        2          35s
elasticsearch-cdm-fyglie0e-2-7dfb646485-s7zhz   0/2     Pending                 0          3s
fluentd-csxfb                                   0/1     Init:CrashLoopBackOff   2          35s
fluentd-jvvd7                                   0/1     Init:0/1                0          35s
fluentd-qbbl7                                   0/1     Init:0/1                1          35s
```

elasticsearch-cdm-fyglie0e-1-5c86bfd95c-q9zwm pod 的日志如下： 
```bash
Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore /etc/elasticsearch//secret/logging-es.jks -destkeystore /etc/elasticsearch//secret/logging-es.jks -deststoretype pkcs12".
Certificate was added to keystore

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore /etc/elasticsearch//secret/logging-es.jks -destkeystore /etc/elasticsearch//secret/logging-es.jks -deststoretype pkcs12".
Certificate was added to keystore
Certificate was added to keystore
cp: cannot create regular file '/etc/elasticsearch/elasticsearch.yml': Permission denied
cp: cannot create regular file '/etc/elasticsearch/log4j2.properties': Permission denied
time="2020-08-25T11:49:33Z" level=info msg="mapping path \"/\" => upstream \"https://localhost:9200/\""
time="2020-08-25T11:49:33Z" level=info msg="HTTPS: listening on [::]:60001"
time="2020-08-25T11:49:33Z" level=info msg="HTTPS: listening on [::]:60000"
```

问题排查：
1. 读写权限问题，第一步想到的肯定是存储问题，发现数据持久化目录是 /elasticsearch/persistent 和上面 /etc/elasticsearch/ 目录不相干， 而且即使改成不使用网络存储还是有这个问题
2. 查看容器镜像属性 podman  inspect registry.example.com:5000/openshift4/ose-logging-elasticsearch6@sha256:88e6b0b164f8f798032b82f97c23c77756eead60a79764d2a15a449f8423c1b6   找到启动参数 /opt/app-root/src/run.sh ，启动脚本里有一句 cp /usr/share/java/elasticsearch/config/* $ES_PATH_CONF   
/usr/share/java/elasticsearch/config/ ，这个通过deployment 和 configmap(elasticsearch) 可以看出，会把新的 elasticsearch.yml 挂载到这个目录下  
$ES_PATH_CONF 这个变量是在初始镜像中定义的 "ES_PATH_CONF=/etc/elasticsearch/"  
所以是在复制替换文件时候出现了权限问题  
3. 我在base 节点使用 podman run -it container_image bash 运行了一个es 容器测试  
发现 /etc/elasticsearch/ 目录权限是 root，而容器用户是 uid=1000(elasticsearch) gid=1000(elasticsearch) 

```bash
bash-4.2$ ls -l /etc/elasticsearch/
total 16
-rw-rw---- 1 root          root 2853 Jul 24 06:51 elasticsearch.yml
-rw-rw---- 1 root          root 3748 Jul 24 06:51 jvm.options
-rw-rw---- 1 root          root 5156 Jul 24 06:51 log4j2.properties
drwxrwsrwx 2 root          root    6 Jul 24 06:51 scripts

bash-4.2$ id
uid=1000(elasticsearch) gid=1000(elasticsearch) groups=1000(elasticsearch)

bash-4.2$ grep cp -R .
./logging:    cp $provided_secret_dir/* $secret_dir/
./run.sh:cp /usr/share/java/elasticsearch/config/* $ES_PATH_CONF
./sgconfig/config.yml:            attr_header_prefix: 'x-ocp-'
```

解决方法：
所以解决方法是把镜像里 /etc/elasticsearch/* 和其他几个目录文件的权限改成和用户一致（uid 1000 gid 1000），不能直接把es的运行用户改成root，启动会报错

step1： 
建个 Dockerfile，修改文件权限，重新推送到仓库  

```bash
mkdir /tmp/es-dockerfile
cd /tmp/es-dockerfile

# Dockerflie 修改权限，需要先切换到root用户，不然没有执行 chown 权限，再切回 1000 uid
# 这边有三个目录的权限需要修改
# cat Dockerfile 
FROM registry.example.com:5000/openshift4/ose-logging-elasticsearch6@sha256:88e6b0b164f8f798032b82f97c23c77756eead60a79764d2a15a449f8423c1b6
USER root
RUN chown -R 1000:1000 /etc/elasticsearch/ && \
    chown -R 1000:1000 /opt/app-root/src/ && \
    chown -R 1000:1000 /usr/share/elasticsearch/
USER 1000

# 制作镜像并推送仓库
podman build -t registry.example.com:5000/openshift4/ose-logging-elasticsearch6:v1 . 
podman push registry.example.com:5000/openshift4/ose-logging-elasticsearch6:v1
``` 

step2:  
在 web console 页面， Operators - Installed Operators，选择 openshift-operators-redhat project，注意是 openshift-operators-redhat 这个 project，进入 Elasticsearch Operator ，在 YAML 页面，修改 ELASTICSEARCH_IMAGE 这个变量，改成 registry.example.com:5000/openshift4/ose-logging-elasticsearch6:v1 ,也就是我们新的镜像

也可以通过命令方式
```bash
# 先查找已安装的 operator，再编辑
~ oc get ClusterServiceVersion -n openshift-operators-redhat
NAME                                           DISPLAY                  VERSION                 REPLACES   PHASE
elasticsearch-operator.4.5.0-202007240519.p0   Elasticsearch Operator   4.5.0-202007240519.p0              Succeeded
~ oc edit ClusterServiceVersion elasticsearch-operator.4.5.0-202007240519.p0 -n openshift-operators-redhat

# 修改 ELASTICSEARCH_IMAGE 这个变量
...
                      - name: ELASTICSEARCH_IMAGE
                        value: >-
                          registry.example.com:5000/openshift4/ose-logging-elasticsearch6:v1

...

# 修改之后会看到 elasticsearch-operator 这个 pod 更新
~ oc get pod -n openshift-operators-redhat
NAME                                      READY   STATUS        RESTARTS   AGE
elasticsearch-operator-5d57f95d6c-vnmfq   1/1     Running       0          7s
elasticsearch-operator-859cf9f876-hkc7q   0/1     Terminating   0          6m15s
```

step3：
在 Installed Operators ，选择project openshift-logging， 选择 operator Cluster Logging, 删除创建的 instance ，并确认es ，fluentd 这些pod 被清除掉。 
只会保留 cluster-logging-operator 这个 pod
```bash
[root@bastion es-dockerfile]# oc get pod -n openshift-logging
NAME                                       READY   STATUS    RESTARTS   AGE
cluster-logging-operator-dc666d47c-dqlzh   1/1     Running   0          8d
```

再重新创建 ClusterLogging   
```bash
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance" 
  namespace: "openshift-logging"
spec:
  managementState: "Managed"  

...
```

之后就是等待所有 pod 创建完成

```bash
[root@bastion es-dockerfile]# oc get pod -n openshift-logging
NAME                                            READY   STATUS    RESTARTS   AGE
cluster-logging-operator-dc666d47c-dqlzh        1/1     Running   0          8d
elasticsearch-cdm-sf8h63d4-1-77fb57df57-6jtrw   2/2     Running   0          3m21s
elasticsearch-cdm-sf8h63d4-2-84865c4ddf-whcpv   2/2     Running   0          2m20s
elasticsearch-cdm-sf8h63d4-3-797c999f7c-pzrhq   2/2     Running   0          79s
fluentd-872lj                                   1/1     Running   0          3m23s
fluentd-gk55j                                   1/1     Running   0          3m23s
fluentd-pkwl4                                   1/1     Running   0          3m23s
kibana-78f94c4945-4gfw5                         2/2     Running   0          3m19s
```

#### issue2 podman build 失败  
podman build 报错 panic: runtime error: invalid memory address or nil pointer dereference

问题现象：
[root@bastion es-dockerfile]# podman build -t registry.example.com:5000/openshift4/ose-logging-elasticsearch6:v1 .
STEP 1: FROM registry.example.com:5000/openshift4/ose-logging-elasticsearch6@sha256:88e6b0b164f8f798032b82f97c23c77756eead60a79764d2a15a449f8423c1b6
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x58 pc=0x5633eec25f18]

解决方法：
升级podman 版本，当前 1.6.4 ，升级到当前最新版本 2.0.5
```bash
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
```

### 参考文档  
https://access.redhat.com/documentation/en-us/openshift_container_platform/4.4/html/logging/cluster-logging-deploying#cluster-logging-deploy-console_cluster-logging-deploying

https://docs.openshift.com/container-platform/4.4/logging/cluster-logging-deploying.html