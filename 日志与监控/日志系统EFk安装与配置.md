## 日志系统EFk安装与配置

本篇介绍openshift4 日志系统的安装与配置。 openshift4 默认安装没有自带日志系统，需要我们手动通过 operator 安装。  
如果集群可以联网，可以直接在 web 页面，operatorhub 页面找到 Cluster logging 进行部署。  
离线环境，需先准备离线的 operatorhub 环境，可参照文章 [离线部署operatorhub并批量导入operator](../应用商店/离线部署operatorhub并批量导入operator.md)

### 简介
集群日志系统基于Elasticsearch，Fluentd和Kibana（EFK）。收集器Fluentd部署到OpenShift集群中的每个节点。它收集所有节点和容器日志，并将它们写入Elasticsearch（ES）。Kibana是集中式的Web UI，用户和管理员可以在其中使用汇总的数据创建丰富的可视化效果和仪表板，数据来源于ES。




### 安装 Cluster logging


查看服务下所有实例的日志  
应用中心--服务管理，选择服务名称进入服务详情，在服务详情页面点击更多，会看到"服务日志"选项  
![deployment-log-1.png](../images/application/deployment-log-1.png)

日志内容，可以看出，两个实例的日志都有

![deployment-log-2.png](../images/application/deployment-log-2.png)

### 1. 查看实例日志
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



### 2. 进入容器终端
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
