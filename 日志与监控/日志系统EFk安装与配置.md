## 日志系统EFk安装与配置

本篇介绍openshift4 日志系统EFK的安装与配置。 openshift4 默认安装没有自带日志系统，需要我们手动通过 operator 安装。  

### EFK 简介
集群日志系统基于Elasticsearch，Fluentd和Kibana（EFK）。收集器Fluentd部署到OpenShift集群中的每个节点。它收集所有节点和容器日志，并将它们写入Elasticsearch（ES）。Kibana是集中式的Web UI，用户和管理员可以在其中使用汇总的数据创建丰富的可视化效果和仪表板，数据来源于ES。





如果集群可以联网，可以直接在 web console， operatorhub 页面部署，需要用到 Elasticsearch Operator和Cluster Logging Operator。  
离线环境，需先准备离线的 operatorhub 环境，可参照文章 [离线部署operatorhub并批量导入operator](../应用商店/离线部署operatorhub并批量导入operator.md)

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


```






![deployment-log-1.png](../images/application/deployment-log-1.png)



https://access.redhat.com/documentation/en-us/openshift_container_platform/4.4/html/logging/cluster-logging-deploying#cluster-logging-deploy-console_cluster-logging-deploying

https://docs.openshift.com/container-platform/4.4/logging/cluster-logging-deploying.html