* [简介](README.md)
* 集群安装与配置管理
  * [在笔记本上运行的OpenShift集群-CRC]](./cluster-install-and-managerment/openshift-crc-on-mac.md) 
  * [4.7 在线安装 DHCP方式](./cluster-install-and-managerment/openshift4.7-install-online-DHCP.md)
  * [4.3 离线部署 DHCP方式](https://github.com/cai11745/k8s-ocp-yaml/blob/master/ocp4/2020-02-25-openshift4.3-install-offline-dhcp.md)
  * [4.4 在线部署 静态IP(支持all-in-one单点)](https://github.com/cai11745/k8s-ocp-yaml/blob/master/ocp4/2020-02-25-openshift4.4-install-online-staticIP-allinone.md)
  * [离线升级4.5->4.6](./cluster-install-and-managerment/offline-upgrade-4.5-to-4.6.md)
  * [AWS公有云安装openshift4](./cluster-install-and-managerment/openshift4-install-on-aws.md)
  * [添加worker node](./cluster-install-and-managerment/add-worker-node.md)


* 权限与认证
  * [访问web控制台](./user-permissions/web-console.md)
  * [自定义web控制台--告示，超链接](./cluster-install-and-managerment/web-console-customization.md)
  * [使用HTPasswd方式管理用户](./user-permissions/add-htpasswd-provider-oauth.md)
  * [移除kubeadmin默认管理员](./user-permissions/remove-kubeadmin.md)

* 存储管理
  * [nfs-provisioner提供storageclass动态存储](./storage/nfs-provisioner-storageclass.md)

* 日志与监控
  * [日志系统EFK安装与配置](./logging-and-monitoring/logging-EFK-install.md)

* 镜像仓库
  * [安装内部image-registry及存储持久化](./image-registry/install-internal-image-registry-and-storage-persistence.md)
  * [使用Samples-Operator更新ImageStream](./image-registry/update-ImageStream-with-Samples-Operator.md)

* 应用商店
  * [搭建私有helm仓库及图形管理界面](./application-store/install-helm-repository-and-UI.md)
  * [离线部署operatorhub并批量导入](./application-store/offline-operatorhub-install.md)

* ServiceMesh
  * [ServiceMesh简介与安装](./ServiceMesh/ServiceMesh-install.md)
  * [部署bookinfo demo](./ServiceMesh/deploy-bookinfo-demo.md)

* DevOps
  * [Pipeline(Tekton)简介与场景示例](./DevOps/openshift-pipeline-Tekton-install.md)
  * [Pipeline(Tekton)调优提速](./DevOps/openshift-pipeline-Tekton-advanced.md)
  * [GitOps持续交付 - ArgoCD](./DevOps/gitops-argocd.md)


* 资源管理--未就绪

* 应用管理--未就绪

----

* [Last part without title]()

**扫一扫，关注微信公众号，不迷路，实时分享容器平台技术。**

<div align="center"><img width="300" height="300" src="https://ftp.rss.ink/imgs/2021/05/11a97ed9fa3b9d85.jpeg"/></div>
