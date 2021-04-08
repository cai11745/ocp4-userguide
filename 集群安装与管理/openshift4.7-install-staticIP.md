---
layout: post
title:  openshift4.7 安装部署 - DHCP online
category: openshift，4.7, install 
description: 
---

本文描述openshift4.7 baremental 在线安装方式，我的环境是 vmwamre esxi 虚拟化，也适用于其他方式提供的虚拟主机或者物理机。

本环境 3mater，2 worker

### 部署环境介绍

#### 本此部署使用资源

方案还是采用高可用，比官方多了一个base节点，用来搭建部署需要的dns，httpd 等服务，这台系统用Centos7.6，因为centos解决源比较方便，等熟悉部署及所需安装包后可以换成RHEL。

其他机器都用RHCOS，就是coreos专门针对openshift的操作系统版本。引导光盘为 rhcos-4.7.0-x86_64-live.x86_64.iso  

|Machine|OS|vCPU|RAM|Storage|IP|
|-|-|-|-|-|-|
|bastion|Centos7.6|2|8GB|100 GB|192.168.2.29|
|bootstrap|RHCOS|4|16GB|120 GB|192.168.2.30|
|master1|RHCOS|4|16 GB|120 GB|192.168.2.31|
|master2|RHCOS|4|16 GB|120 GB|192.168.2.32|
|master3|RHCOS|4|16 GB|120 GB|192.168.2.33|
|worker-0|RHCOS|4|16 GB|120 GB|192.168.2.34|
|worker-1|RHCOS|4|16 GB|120 GB|192.168.2.35|

节点角色：  
1台 基础服务节点，用于安装部署所需的 dns，httpd服务。系统不限。同时承担负载均衡，用于负载master节点api及router，功能同3.x。这个服务现在需要自己部署，生产环境也可用硬负载。  
1台 部署引导节点 Bootstrap，用于安装openshift集群，在集群安装完成后可以删除。系统RHCOS  
3台 控制节点 Control plane，即master，通常使用三台部署高可用，etcd也部署在上面。系统RHCOS  
2台 计算节点 Compute，用于运行openshift基础组件及应用 。系统RHCOS

### 安装顺序

顺序就是先准备基础节点，包括需要的 dns、dhcp 文件服务器、引导文件等，然后安装引导机 bootstrap，再后面就是 master， 再 node

### 安装准备
#### 安装准备 - base基础配置
|base|centos7.6|4|8GB|100 GB|192.168.2.29|

1. 安装系统 centos7.6 mini  
设置IP，设置主机名，关闭防火墙和selinux
注意所有节点主机名采用三级域名格式  如 master1.aa.bb.com

```bash
hostnamectl set-hostname bastion.ocp4.example.com
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
systemctl disable firewalld
systemctl stop firewalld

```
2. 设置 yum 缓存包到本地 - 可选步骤  
针对在线安装，这样以后如果切换到离线安装，可以方便知道用了哪些包  
编辑 /etc/yum.conf 把keepcache改为1  
keepcache=1

```bash
yum -y install podman httpd httpd-tools vim

```

#### 安装准备 oc command
   
下载最新的oc openshift-install 命令。进入下载链接，下载openshift client，openshift-install  
直接选一个4.7最新的 

https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/
 
```bash
tar -zxvf openshift-client-linux-4.7.5.tar.gz
mv oc /usr/local/bin/
oc version
Client Version: 4.7.5
```

```bash
tar -zxvf openshift-install-linux-4.7.5.tar.gz
chmod +x openshift-install
mv openshift-install /usr/local/bin/
openshift-install version

```

#### 安装准备 -- DNS，DHCP 部署

部署dnsmasq服务，这个服务可以同时做dns server, dhcp server和tftp，为后面的 RHCOS 提供PXE自动安装

```bash
yum install dnsmasq bind-utils tftp-server ipxe-bootimgs -y
# 配置dnsmasq，配置文件如下
# 需要提前把RHCOS节点的虚机创建好，在虚机配置里把MAC地址记下来
# 因为在内网里有其他机器，不把MAC和IP绑定，DHCP server准备的IP容易被其他机器拿去。

cd /etc/dnsmasq.d/
 
vi ocp4.conf
dhcp-range=192.168.2.30,192.168.2.38,255.255.255.0
enable-tftp 
tftp-root=/var/lib/tftpboot
dhcp-match=set:bios,option:client-arch,0
dhcp-boot=tag:bios,undionly.kpxe
dhcp-match=set:efi32,option:client-arch,6
dhcp-boot=tag:efi32,ipxe.efi
dhcp-match=set:efibc,option:client-arch,7
dhcp-boot=tag:efibc,ipxe.efi
dhcp-match=set:efi64,option:client-arch,9
dhcp-boot=tag:efi64,ipxe.efi
dhcp-userclass=set:ipxe,iPXE
dhcp-option=option:router,192.168.2.1
dhcp-option=option:dns-server,192.168.2.29
dhcp-boot=tag:ipxe,http://bastion.ocp4.example.com:8080/boot.ipxe

address=/bastion.ocp4.example.com/192.168.2.29
address=/api.ocp4.example.com/192.168.2.29
address=/apps.ocp4.example.com/192.168.2.29
address=/api-int.ocp4.example.com/192.168.2.29
address=/master1.ocp4.example.com/192.168.2.31
address=/etcd-0.ocp4.example.com/192.168.2.31
address=/master2.ocp4.example.com/192.168.2.32
address=/etcd-1.ocp4.example.com/192.168.2.32
address=/master3.ocp4.example.com/192.168.2.33
address=/etcd-2.ocp4.example.com/192.168.2.33
address=/bootstrap.ocp4.example.com/192.168.2.30
address=/registry.example.com/192.168.2.29
srv-host=_etcd-server-ssl._tcp.ocp4.example.com,etcd-0.ocp4.example.com,2380,10
srv-host=_etcd-server-ssl._tcp.ocp4.example.com,etcd-1.ocp4.example.com,2380,10
srv-host=_etcd-server-ssl._tcp.ocp4.example.com,etcd-2.ocp4.example.com,2380,10

dhcp-host=00:50:56:95:45:00,bootstrap.ocp4.example.com,192.168.2.30
dhcp-host=00:50:56:95:69:72,master1.ocp4.example.com,192.168.2.31
dhcp-host=00:50:56:95:26:f2,master2.ocp4.example.com,192.168.2.32
dhcp-host=00:50:56:95:25:8d,master3.ocp4.example.com,192.168.2.33
dhcp-host=00:50:56:95:5d:13,worker1.ocp4.example.com,192.168.2.34
dhcp-host=00:50:56:95:fc:41,worker2.ocp4.example.com,192.168.2.35

log-queries
log-dhcp
log-facility=/var/log/dnsmasq.log 

```

dhcp-range: 几台RHCOS预留的IP池，及掩码  
dhcp-option=option:router: 网关地址  
dhcp-option=option:dns-server: 这台部署机的IP，因为这台做dns server  
dhcp-boot=tag:ipxe: 写这台部署机的主机名，其他不用改  
下面的address，分别是主机名和IP地址  
api.ocp4.example.com 多master，指向master的负载，这边写部署机，后面部署机上面安装负载均衡，负载指向到三个master api  
apps.ocp4.example.com 指向部署机，部署机会安装通过haproxy 负载到infra的router  
api-int.ocp4.example.com 同api.ocp4.example.com,指向openshift api,就是当前bastion节点  
dhcp-host dhcp与mac配置，分别是几台RHCOS的mac地址，主机名，ip地址

配置tftp server，并启动dnsmasq，如果本机开启了防火墙还需要防火墙放行，我这台上面已经关闭防火墙了  

```bash
mkdir -p /var/lib/tftpboot
ln -s /usr/share/ipxe/undionly.kpxe /var/lib/tftpboot

# 启动dnsmasq
systemctl enable dnsmasq && systemctl restart dnsmasq

# 查看dnsmasq服务状态，如果有报错，会提示错误在配置文件哪一行
systemctl status dnsmasq -l

# 本地修改 resolv.conf 文件
# 加一行下面参数，IP是本机地址，要写在其他nameserver前面
# 注意这个参数写在这里重启应该会丢，要想不丢，修改  /etc/sysconfig/network-scripts/ifcfg-ens192
# 加一行 DNS1=192.168.2.29 ,然后systemctl restart network
vi /etc/resolv.conf
nameserver 192.168.2.29


# 验证dns server
yum install bind-utils -y

# 测试下dns解析
nslookup master1.ocp4.example.com & nslookup bootstrap.ocp4.example.com

# 测试结果，能正常解析IP
Name:   bootstrap.ocp4.example.com
Address: 192.168.2.30
Name:   master1.ocp4.example.com
Address: 192.168.2.31
```



#### 安装准备 -- haproxy 负载均衡

单master可以跳过这一步，不过我还没测试单master部署。

haproxy主要用于负载master api 6443 ，worker节点的router 80 443，可以被其他负载代替

```bash
yum install haproxy -y
cd /etc/haproxy/
vim haproxy.cfg  #在最下面加配置文件，也可以把自带的frontend 和backend删掉，没有用

# 可选项,可以通过页面查看负载监控状态
listen stats
    bind :9000
    mode http
    stats enable
    stats uri /
    monitor-uri /healthz

# 负载master api，bootstrap 后面删掉
frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
     server bootstrap 192.168.2.30:6443 check
    server master1 192.168.2.31:6443 check
    server master2 192.168.2.32:6443 check
    server master3 192.168.2.33:6443 check

frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
     server bootstrap 192.168.2.30:22623 check
    server master1 192.168.2.31:22623 check
    server master2 192.168.2.32:22623 check
    server master3 192.168.2.33:22623 check

# 负载router，就是*.apps.ocp4.example.com , 这个域名如果 dns server 指向了本机，则这边必须配置，否则对于测试环境可选项。
frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server worker0 192.168.2.34:80 check
    server worker1 192.168.2.35:80 check
    server worker2 192.168.2.36:80 check

frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server worker0 192.168.2.34:443 check
    server worker1 192.168.2.35:443 check
    server worker2 192.168.2.36:443 check
```

启动服务

```bash
systemctl enable haproxy && systemctl start haproxy
```

验证服务

通过浏览器页面查看 IP:9000 可以看到haproxy的监控页面，当前后端服务还没起，所以很多红色的。


#### 安装准备 -- matchbox部署

安装配置matchbox ,该服务主要是用作pxe安装rhcos时,分配安装文件

文件目录示意，下面会说明文件如何获取配置  

```bash
[root@bastion ~]# tree /var/lib/matchbox/
/var/lib/matchbox/
├── assets
│   ├── rhcos-4.7.0-x86_64-live-initramfs.x86_64.img
│   ├── rhcos-4.7.0-x86_64-live-kernel-x86_64
│   └── rhcos-4.7.0-x86_64-metal.x86_64.raw.gz
├── groups
│   ├── bootstrap.json
│   ├── master1.json
│   ├── master2.json
│   ├── master3.json
│   ├── worker1.json
│   └── worker2.json
├── ignition
│   ├── bootstrap.ign
│   ├── master.ign
│   └── worker.ign
└── profiles
    ├── bootstrap.json
    ├── cptnod.json
    ├── infnod.json
    └── master.json

4 directories, 16 files
```

安装 matchbox-v0.8.3-linux-amd64.tar.gz,可以到github的release页面下载, 
https://github.com/poseidon/matchbox/releases
 
```bash
tar xf matchbox-v0.8.3-linux-amd64.tar.gz
cd matchbox-v0.8.3-linux-amd64/
cp matchbox /usr/local/bin
cp contrib/systemd/matchbox.service /etc/systemd/system/matchbox.service
vi /etc/systemd/system/matchbox.service  
# 把 User Group 两行删掉，直接以root运行
mkdir -p /var/lib/matchbox/{assets,groups,ignition,profiles}
```

下载rhcos 安装文件，本次部署需要下面这三个，可以从下面官网连接下载，rhcos的版本不会每次都随着ocp版本升级，rhcos的版本不能高于ocp版本  
https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos

kernel: rhcos-<version>-live-kernel-<architecture>
initramfs: rhcos-<version>-live-initramfs.<architecture>.img
rootfs: rhcos-<version>-live-rootfs.<architecture>.img

这三个文件放到 /var/lib/matchbox/assets 目录下

把离线安装包里 matchbox 目录下的 groups profiles 挪到 /var/lib/matchbox/

进入 /var/lib/matchbox/ 目录，分别修改 profiles 和 groups 下的文件， ignition 的文件后面会通过命令生成。

文件参考
https://github.com/cai11745/k8s-ocp-yaml/tree/master/ocp4/config-files/matchbox

profiles 目录下修改配置文件，主要确认kernel initrd 对应的配置文件名称及 url , 
install_dev ：在不同虚拟化或者物理平台磁盘命名可能不一样，看下当前部署机是sda 还是 vda，rhcos下命名规则应该是一样的

groups 目录下修改节点配置文件，主要是mac地址和检查文件里的 profiles 参数是否正确


启动matchbox 
```bash

chmod 755 -R /var/lib/matchbox/

systemctl enable matchbox
systemctl restart matchbox
```

matchbox 监听8080 端口，可以通过浏览器访问到 http://192.168.2.29:8080/assets/


### openshift 安装

#### 准备 install-config.yaml

首先创建 install-config.yaml 文件

准备拉取镜像权限认证文件。 
从 Red Hat OpenShift Cluster Manager 站点的 Pull Secret 页面下载 registry.redhat.io 的 pull secret。
https://cloud.redhat.com/openshift/install/pull-secret


创建一对公钥和私钥，用作登录RHCOS  
```bash
ssh-keygen -t rsa -b 2048 -N "" -f /root/.ssh/id_rsa

mkdir /opt/install 

cd /opt/install 
vi install-config.yaml

apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetworks:
  - cidr: 10.254.0.0/16
    hostPrefix: 24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"registry.example.com:5000": {"auth": "cm9vdDpwYXNzd29yZA==","email": "you@example.com"}}}'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCUczW55sl8KWZj4CcsHqfF2pc3vUmw8EXvACbnGy4QzgvtbSe96ePsAIBxt/xEx0YuqP9PABWzm/qdVZvVDwK3vyIgtA0avb8/Mch+XE2qV9ftENAKwD/vXE9oU4Riic3nOzRIyygKwPU5kXqvRnEyUvKStCyadPq7/jcS5VcZM7QXX//PGKZGSDFOsthscGO9g8vcS7AdsvsGwkuEc7g9d6u826cXF/7r2fJMMdHJDSQmrDMxEOU3ncHZ7Noym2jVpOq50t1aXXC5R1LLK+vSpb48LR4Dsnwa3g+jX3dqxTHmlRAH789iBEyxNfIsw2iJpPxJZE8nb7WwN7DZMglV root@bastion.ocp4.example.com'


```
参数解读：
1. baseDomain: 就是上面每个主机配置的域名  
2. metadata.name: 就是节点master/worker名称后面一级，这也是为什么主机名要用好几级  
3. worker: 0 不代表0个worker，此处不用改  
4. cidr: 是pod ip池 ，servicenetwork 就是k8s里的service ip，两个都要和虚拟机本地ip网段区分开  
5. pullSecret: 私有仓库的地址和用户密码的base64，就是redhat网站下载的pull-secret.json，内容两头需要加上单引号
6. sshKey: ssh公钥，用于后面免密登录 cat /root/.ssh/id_rsa.pub 全部内容贴进去


#### 生成 ign 配置文件

```bash
# 备份下文件，下面命令执行后文件会被自动删除
cp install-config.yaml /tmp/install-config.yaml.bak 

# 生成ign文件，如果要重新生成，要把整个ocp4目录删了重来
# 有个隐藏文件 .openshift_install_state.json 会导致报错  
openshift-install create manifests --dir=./

# 为了让master节点上不允许运行Openshift之外其他pod，需要修改cluster-scheduler-02-config.yml文件  
vim manifests/cluster-scheduler-02-config.yml   
将参数”mastersSchedulable”设置为 false

# 生成引导配置
openshift-install create ignition-configs --dir=./


# 查看文件目录
[root@bastion ocp4]# tree
.
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap.ign
├── master.ign
├── metadata.json
└── worker.ign

# 把 .ign 文件复制到matchbox 数据目录
cp *.ign /var/lib/matchbox/ignition/

```

### openshift 安装

部署机 bastion 执行命令监控部署状态，等master 都装完，这条命令会自动停止。  
```bash
cd ~
openshift-install --dir=/opt/install wait-for bootstrap-complete --log-level debug
```
安装顺序是 bootstrap-> master -> worker  
前一角色完成后，再安装下一角色  

#### 安装 bootstarp 

把bootstrap虚拟打开，第一次启动的时候找不到本地系统盘会自动切换到网卡pxe启动。如果没有从pxe启动，进入bios 修改下启动顺序，改成从网络启动  

待系统安装完成后，bootstrap虚机控制台会停留在登录页面。

在部署机 base 可以通过 ssh 命令进入bootstrap

```bash
ssh core@172.28.105.18

# 查看运行容器，sudo podman ps, 应该有运行中容器
# 如果没有在运行的容器， ps -ef 看下是否有podman pull 的进程。异常可参照最下方 Troubleshooting
sudo podman ps 
 
# 检查方法:
# 查看网络信息，地址和dns 配置无误
ip route show
cat /etc/resolv.conf

# 查看进程端口，要求 6443 22623 端口已启动
[core@bootstrap openshift]$ sudo netstat -ltnp|grep 22623
tcp6       0      0 :::22623                :::*                    LISTEN      3999/machine-config 
[core@bootstrap openshift]$ 
[core@bootstrap openshift]$ sudo netstat -ltnp|grep 6443
tcp6       0      0 :::6443                 :::*                    LISTEN      3833/kube-etcd-sign 

# 以上端口无误，则持续观察 bootkube 服务状态，然后继续安装 master 节点
journalctl -b -f -u bootkube.service

# 也可以通过haproxy的页面，就是172.28.105.11:9000 可以看到bootstrap的状态变成了绿色
# 说明这个时候bootstrap 已经部署成功
```

**故障诊断思路**  
bootstrap 节点先查看是否有镜像和容器  
若 sudo podman images 看不到镜像，参考 Troubleshooting #1  
有镜像和运行pod，但是pod 数量不对，查看pod 日志，及通过 journalctl -f 命令查看日志输出

#### master 部署  
把master几台虚机打开，会自动从pxe启动部署，检查方法与上面类似  
通过haproxy页面查看master节点api及machine-config状态  
以及通过netstat 查看端口 api  6443， etcd 2379  

部署完成后，从haproxy上看到master-1的api状态异常，master-0 master-2 正常，还不太熟悉服务组件运行方式，我重启master-1后正常了，后续再看。

oc 命令可以看到master状态  
```bash
 cp /opt/install/auth/kubeconfig ~/.kube/config  
[root@bastion profiles]# oc get node
NAME      STATUS   ROLES    AGE   VERSION
master1   Ready    master   26m   v1.20.0+bafe72f
master2   Ready    master   25m   v1.20.0+bafe72f
master3   Ready    master   24m   v1.20.0+bafe72f
```

在bastion 节点，haproxy的配置文件，backend 中的 bootstrap 需要删掉，并重启haproxy。

```bash
vi /etc/haproxy/haproxy.cfg
 backend openshift-api-server 删除bootstrap这行      server bootstrap 192.168.2.11:6443 check

 backend machine-config-server 删除bootstrap这行       server bootstrap 192.168.2.11:22623 check

# 重启服务
systemctl restart haproxy
```

#### 安装 worker

部署机 bastion 执行命令监控部署状态  
```bash
cd ~
openshift-install --dir=/root/ocp4 wait-for install-complete --log-level debug
```

当两台worker 在控制台看到已经部署完

在部署机之前 get csr 命令，查看node 节点加入申请，批准之，然后就看到了node节点。 大功告成！！！

```bash
[root@bastion ~]# oc get csr 
NAME        AGE   REQUESTOR                                                                   CONDITION
csr-2mxfw   13m   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-p6ckt   13m   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending

[root@bastion ~]# oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve

[root@bastion profiles]# oc get node
NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   40m     v1.20.0+bafe72f
master2   Ready    master   39m     v1.20.0+bafe72f
master3   Ready    master   39m     v1.20.0+bafe72f
worker1   Ready    worker   5m18s   v1.20.0+bafe72f
worker2   Ready    worker   5m17s   v1.20.0+bafe72f

```


#### Web console 登录

ocp4的web console 入口走router了，所以找下域名  
首先找到我们的域名，然后在我们自己电脑上 hosts添加解析，指向到 basetion，因为basetion 节点做了负载均衡把 80 443 端口负载到了两台worker，这样就能够访问openshift 的web 控制台了

```bash
[root@bastion ~]# oc get route --all-namespaces |grep console-openshift
openshift-console          console             console-openshift-console.apps.example.com                       console             https   reencrypt/Redirect     None
```

把这条写入hosts  
192.168.2.29 oauth-openshift.apps.example.com console-openshift-console.apps.example.com

然后浏览器访问console  
https://console-openshift-console.apps.example.com

第一次发现页面打不开. Application is not available , 这是router返回的，说明router服务好使，web console 服务异常。

```bash
[root@bastion ~]# oc get pod --all-namespaces |grep console
openshift-console-operator                              console-operator-597c74c496-gr78z                                 1/1     Running                    0          109m
openshift-console                                       console-5f4b9f69c7-kmtlz                                          0/1     Pending                    0          41m
openshift-console                                       console-87c8d8ddc-2tv59                                           0/1     Pending                    0          41m
openshift-console                                       console-87c8d8ddc-k2kpl                                           0/1     Pending                    0          37m
openshift-console                                       console-87c8d8ddc-zdpc2                                           0/1     UnexpectedAdmissionError   0          41m
openshift-console                                       downloads-7fdfb77b95-bk9cr                                        1/1     Running                    0          109m
openshift-console                                       downloads-7fdfb77b95-k84hj                                        1/1     Running                    0          109m
```

嗯，pod pending， 看event, 是通过node selector 指定了master，两个node的标签不匹配，但是master的cpu又不够。所以给三台master虚拟修改下CPU，之前是2核改成4，重启下系统

```bash
[root@bastion ~]# oc -n openshift-console describe po console-5f4b9f69c7-kmtlz

  Warning  FailedScheduling  <unknown>  default-scheduler  0/5 nodes are available: 2 node(s) didn't match node selector, 3 Insufficient cpu.

```

重新看下 console pod，正常后console 就可以打开了  
`oc -n openshift-console  get pod` 

用户名是 kubeadmin
密码在这个文件里
cat /root/ocp4/auth/kubeadmin-password

### Troubleshooting

#### 1. bootstrap 节点没有任何镜像和容器

sudo podman images /sudo podman ps -a 看不到任何镜像或容器，通过 journalctl -f 命令发现了如下异常

Jan 10 00:28:49 localhost release-image-download.sh[1201]: Error: error pulling image "quay.io/openshift-release-dev/ocp-release@sha256:6c57b48ec03382d9d63c529d4665d133969573966400515777f36dd592ad834a": unable to pull quay.io/openshift-release-dev/ocp-release@sha256:6c57b48ec03382d9d63c529d4665d133969573966400515777f36dd592ad834a: unable to pull image: Error initializing source docker://quay.io/openshift-release-dev/ocp-release@sha256:6c57b48ec03382d9d63c529d4665d133969573966400515777f36dd592ad834a: pinging docker registry returned: Get https://quay.io/v2/: dial tcp: lookup quay.io on 172.28.105.20:53: server misbehaving

首先排查 bootstrap 节点 mirror 配置有没有指向到内部仓库，sudo cat /etc/containers/registries.conf ， 发现是有 mirror 配置的

应该是 registry.example.com 没有连上，又重新指向到了 quay.io

在 base 节点本地测试下 pull， podman pull registry.example.com:5000/ocp4/openshift4@sha256:6c57b48ec03382d9d63c529d4665d133969573966400515777f36dd592ad834a 发现是正常的，说明仓库里有这个镜像

在 bootstrap 执行 sudo podman pull  quay.io/openshift-release-dev/ocp-release@sha256:6c57b48ec03382d9d63c529d4665d133969573966400515777f36dd592ad834a 都是返回上面那个报错

说明连接仓库有问题， telnet测试 172.28.105.20 5000 端口通

使用 curl 命令测试接口却没有返回，说明仓库对外服务有异常  
curl -u root:password -k https://registry.example.com:5000/v2/_catalog

在 base 节点重启镜像仓库服务后，还是异常，后把仓库服务的容器删除重建后，可以在 bootstrap 节点正常 pull 镜像

正式环境中，可以考虑使用 harbor 代替 registry 服务，也能更好的解决权限和高可用的问题，harbor 自带高可用方案，可主备自动同步。

#### 2. bootstrap pod 数量不对，Unable to connect to the server: x509: certificate has expired or is not yet valid
bootstrp 节点只能看到两个pod，通过 journalctl -f 命令查看到 x509 的异常

也可以通过 sudo podman ps -a 查看所有pod，包括已经因为异常终止的

```bash
[core@localhost ~]$ sudo podman ps
CONTAINER ID  IMAGE                                                                                                                   COMMAND               CREATED        STATUS            PORTS  NAMES
b18066cdb052  quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:cd0cfc645c901c48c4d5413317698a7e4d3f4d9b84f27d293cc0890137422560  etcdctl --dial-ti...  3 minutes ago  Up 3 minutes ago         etcdctl
056c4180323d  quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:c082e8252484a9dc7a71446ed1afb1b04b465805ed46d38aa3083da6a9b2a983  /usr/bin/kube-etc...  8 hours ago    Up 8 hours ago           etcd-signer

[core@localhost ~]$ journalctl -f

Jan 11 05:47:49 localhost openshift.sh[10101]: error: unable to recognize "./99_kubeadmin-password-secret.yaml": Get https://localhost:6443/api?timeout=32s: x509: certificate has expired or is not yet valid
Jan 11 05:47:49 localhost openshift.sh[10101]: kubectl create --filename ./99_kubeadmin-password-secret.yaml failed. Retrying in 5 seconds...
Jan 11 05:47:50 localhost approve-csr.sh[10103]: Unable to connect to the server: x509: certificate has expired or is not yet valid

```

这个是因为 base 节点生成的安装证书只有24小时有效期，需要重新生成，并引导安装

注意此处要把先备份下文件，然后把目录都干掉，重新创建。


```bash
cp /opt/install/bios.raw.gz /tmp/
rm -rf /opt/install/
mkdir /opt/install
cd /opt/install

# 把备份文件拿回来
cp /tmp/bios.raw.gz .
cp /tmp/install-config.yaml.bak install-config.yaml

# 然后重新生成配置文件
openshift-install create manifests --dir=./
vim manifests/cluster-scheduler-02-config.yml
# 把   mastersSchedulable: true 改为 false

openshift-install create ignition-configs --dir=./


```

#### 参考链接：  
https://access.redhat.com/documentation/zh-cn/openshift_container_platform/4.7/html-single/installing/index#ipi-install-troubleshooting-cluster-nodes-will-not-pxe_ipi-install-troubleshooting
