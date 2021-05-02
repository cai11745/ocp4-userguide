## openshift4 添加 worker node
添加 worker 节点，在 openshift 平台的长期使用中，必然会遇到的场景。

我的环境在部署初期受资源限制，只采用了三个 master 节点，同时担任 worker 角色。后来虚拟化平台加了内存，准备添加三个 worker 节点，做成 3 master， 3 worker 的标准规模测试集群。

如果在初次安装机器的24小时之内，添加 worker 节点，和安装 master/worker 类似，只需要按照集群部署文档来做就行了。

在集群安装完成24小时之后，添加节点，步骤有些不一样，需要重新生成证书。

### 部署准备
我的集群采用的是静态IP安装方式，需要每个master/worker 部署时候手动输入一串信息，包含 ip,dnsserver,配置文件网络路径等。  
bastion 节点部署了 httpd 用于文件服务器，存放 master/worker 部署所需配置文件 .ign 及 rhcos-4.x-x86_64-metal.x86_64.raw.gz

从 api-int.${domain}:22623 获取证书存在本地，domain 在我的环境里是 ocp4.example.com 。可以通过 oc cluster-info 命令获取 api 的地址，api 字段后面的就是

```bash
oc cluster-info 
#  Kubernetes master is running at https://api.ocp4.example.com:6443

# 以下操作在 bastion 节点的集群安装目录，就是有 master.ign worker.ign 那个目录
domain=ocp4.example.com
URL=api-int.${domain}
openssl s_client -connect ${URL}:22623 -showcerts </dev/null 2>/dev/null|openssl x509 -outform PEM > api-int.pem

// api-int.pem 以 BEGIN CERTIFICATE 开头，以 END CERTIFICATE 结尾。
```

转成base64
```bash
base64 --wrap=0 ./api-int.pem 1> ./api.int.base64
```

备份下之前的 worker.ign 文件
```bash
cp ./worker.ign ./worker.ign.backup
```

编辑 worker.ign ，把其中 base64 后面的内容替换成上面 api.int.base64 

确认 worker.ign 可以通过http server 访问到，浏览器访问测试下，我的http 配置的是8080服务端口。
http://192.168.2.29:8080/install/worker.ign


### 安装 worker 节点
下面的内容就和 openshift 初次安装时候是一样的。
首先虚拟机挂载 rhcos-4.5.6-x86_64-installer.x86_64.iso  

在安装界面 "Install RHEL CoreOS" ， 按 Tab 键修改启动参数。 
在 coreos.inst = yes 之后添加。仔细校对参数，不能粘贴

ip=192.168.2.34::192.168.2.1:255.255.255.0:worker0.ocp4.example.com:ens192:none nameserver=192.168.2.29 coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.2.29:8080/install/rhcos.raw.gz coreos.inst.ignition_url=http://192.168.2.29:8080/install/worker.ign 

ip=.. 对应的参数是 ip=ipaddr::gateway:netmask:hostnameFQDN:网卡名称:是否开启dhcp

网卡名称和磁盘名称参照base节点，一样的命名规则，后面两个http文件先在base节点 wget 测试下能否下载

仔细检查，出错了会进入shell界面，可以排查问题。然后重启再输入一次

worker节点部署完成后，在部署机 bastion 执行 oc get csr 命令，查看node 节点加入申请，批准之，然后就看到了node节点。 大功告成！！！

```bash
[root@bastion install]# oc get csr
NAME        AGE   SIGNERNAME                                    REQUESTOR                                                                   CONDITION
csr-9gxpd   28s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
[root@bastion install]# oc adm certificate approve csr-9gxpd 
certificatesigningrequest.certificates.k8s.io/csr-9gxpd approved

这边有一个批量审批的方法：
[root@bastion opt]# yum install epel-release
[root@bastion opt]# yum install jq -y
[root@bastion opt]# oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve

刚加进来状态是not ready，等一会就变成ready了
[root@bastion install]# oc get node
NAME                       STATUS     ROLES           AGE   VERSION
master0.ocp4.example.com   Ready      master,worker   26d   v1.18.3+2cf11e2
master1.ocp4.example.com   Ready      master,worker   26d   v1.18.3+2cf11e2
master2.ocp4.example.com   Ready      master,worker   26d   v1.18.3+2cf11e2
worker0.ocp4.example.com   NotReady   worker          3s    v1.18.3+2cf11e2
```

其他节点同样方法加入，都完成之后是这样，所有节点状态都要是Ready
```bash
[root@bastion install]# oc get node
NAME                       STATUS   ROLES           AGE     VERSION
master0.ocp4.example.com   Ready    master,worker   26d     v1.18.3+2cf11e2
master1.ocp4.example.com   Ready    master,worker   26d     v1.18.3+2cf11e2
master2.ocp4.example.com   Ready    master,worker   26d     v1.18.3+2cf11e2
worker0.ocp4.example.com   Ready    worker          36m     v1.18.3+2cf11e2
worker1.ocp4.example.com   Ready    worker          3m42s   v1.18.3+2cf11e2
worker2.ocp4.example.com   Ready    worker          15m     v1.18.3+2cf11e2
```

### 移除 master 节点调度
由于安装的时候，install-config 文件中，compute 写的 0 ，所以部署之后会默认把 master 节点的调度打开，现在需要把他关闭。

install-config 可以在这里查到。
```bash
oc get cm cluster-config-v1 -n kube-system -o yaml
```

所以需要修改 scheduler 资源对象， 把 mastersSchedulable 参数改为 false
```bash
 # oc edit scheduler
...
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  creationTimestamp: null
  name: cluster
spec:
  mastersSchedulable: false >>> Change this to false
  policy:
    name: ""
status: {}

```

master 节点的 worker ROLE 已经移除
```bash
[root@bastion install]# oc get node
NAME                       STATUS   ROLES    AGE   VERSION
master0.ocp4.example.com   Ready    master   26d   v1.18.3+2cf11e2
master1.ocp4.example.com   Ready    master   26d   v1.18.3+2cf11e2
master2.ocp4.example.com   Ready    master   26d   v1.18.3+2cf11e2
worker0.ocp4.example.com   Ready    worker   68m   v1.18.3+2cf11e2
worker1.ocp4.example.com   Ready    worker   35m   v1.18.3+2cf11e2
worker2.ocp4.example.com   Ready    worker   47m   v1.18.3+2cf11e2
```

其他命令：节点设为不可调度/可调度，驱逐pod
```bash
# 将节点标记为不可调度
oc adm cordon <node1>

# 将节点标记为可调度
oc adm uncordon <node1>

# 驱逐节点上的pod
oc adm drain <node1> <node2> [--pod-selector=<pod_selector>]

# 使用 --force 选项强制删除裸Pod 。设置 true 为时，即使某些Pod不受副本控制器，ReplicaSet，job，daemonset或StatefulSet的管理，也会继续删除：
oc adm drain <node1> <node2> --force=true

# 忽略 daemonset
oc adm drain <node1> <node2> --ignore-daemonsets=true

# 使用--dry-run  列出将要迁移的对象，而不会实际执行迁移
oc adm drain <node1> <node2>  --dry-run=true
```

参考文档：
https://access.redhat.com/solutions/4799921
