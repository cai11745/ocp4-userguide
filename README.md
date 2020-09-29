### openshift4 使用与运维手册

刚开始写，平台基本使用和运维内容都会有。基于4.4 同样适用其他 4.x 差异不大。

---

目录
* 集群安装与配置管理
  * [4.3 离线部署 DHCP方式](https://github.com/cai11745/k8s-ocp-yaml/blob/master/ocp4/2020-02-25-openshift4.3-install-offline-dhcp.md)
  * [4.4 在线部署 静态IP(支持all-in-one单点)](https://github.com/cai11745/k8s-ocp-yaml/blob/master/ocp4/2020-02-25-openshift4.4-install-online-staticIP-allinone.md)
  * [AWS公有云安装openshift4](./集群安装与管理/使用redhat-lab在aws上安装openshift4.md)


* 用户与权限管理
  * [访问web控制台](./用户与权限管理/访问web控制台.md)
  * [使用HTPasswd方式管理用户](./用户与权限管理/使用HTPasswd方式管理用户.md)
  * [移除kubeadmin默认管理员](./用户与权限管理/移除kubeadmin默认管理员.md)

* 存储管理
  * [nfs-provisioner提供storageclass动态存储](./存储管理/nfs-provisioner提供storageclass动态存储.md)

* 日志与监控
  * [日志系统EFK安装与配置](./日志与监控/日志系统EFK安装与配置.md)

* 镜像仓库
  * [安装内部镜像仓库及存储持久化](./镜像仓库/安装内部镜像仓库及存储持久化.md)
  * [使用Samples-Operator更新ImageStream](./镜像仓库/使用Samples-Operator更新ImageStream.md)

* 应用商店
  * [搭建私有helm仓库及图形管理界面](./应用商店/搭建私有helm仓库及图形管理界面.md)
  * [离线部署operatorhub并批量导入](./应用商店/离线部署operatorhub并批量导入operator.md)

* 服务网格ServiceMesh
  * [ServiceMesh简介与安装](./服务网格ServiceMesh/ServiceMesh简介与istio安装.md)
  
* 资源管理--未就绪

* 应用管理--未就绪

* devops



#### 联系方式
* 邮箱: 3162003@qq.com
* QQ  : 3162003

有问题可以联系我或者github提issue，欢迎共同完善。  


**扫一扫，关注微信公众号，不迷路，实时分享容器平台技术。或者直接搜 老菜**

<div align="center"><img width="300" height="300" src="./images/gongzhonghao.jpeg"/></div>
