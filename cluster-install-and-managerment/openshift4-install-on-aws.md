## AWS公有云安装openshift4
本节将使用使用 AWS 环境自动化安装一个高可用的openshift4。

前提：具备AWS账号（好像暂不支持中国区）
我这边是使用的  OPENTLC lab，由红帽提供给合作伙伴的练习环境。  
参照内容： redhat partner center 学习平台 - openshift4 install。
https://training-lms.redhat.com/lmt/clmsbrowsev2.prMain?in_sessionid=155AAA0JA8A3081A

### 环境准备
在 opentlc 练习平台准备基础环境。
 Services → Catalogs → All Services → OPENTLC OpenShift 4 Labs → OpenShift 4 Installation Lab. 点击 order， region 按需，然后点 submit，初始环境准备大概半小时，注意邮件，会将虚机登陆信息发到邮箱。

注意： order 之后不要选择 App Control → Start ，order 之后开始自动准备环境，点击start 可能破坏环境或者出现其他异常。

之后会收到三个邮件，
第一个说你的环境已经开始准备了，第二是虚机准备好了，在配置，巴拉巴拉～
主要是第三个，包含了连接环境的配置。泛域名的地址，aws的key，虚机的ssh 密码。 GUID 是虚机名称 bastion.XXXX.sandbxXXX.opentlc.com 中 bastion 后面的4个字符

这台虚机在闲置8小时后会自动关闭，开启方法：
OPENTLC环境，App Control - Start 选择yes，再submit 启动。

测试ssh连接及查看GUID是否正确，opentlc 在 myservices 里面也能看到 GUID 信息：
```bash
ssh <OPENTLC User Name>@bastion.<GUID>.<TOP LEVEL DOMAIN>
echo $GUID
c3po
```

### 节点配置准备
配置 bastion 节点及准备 aws 工具，openshift install 工具。 
#### 确认GUID与安装aws工具

```bash
# 切换到 root 用户再次确认 GUID 是否正确
sudo -i
echo ${GUID}

# 安装 AWS CLI tools
# Download the latest AWS Command Line Interface
curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
unzip awscli-bundle.zip

# Install the AWS CLI into /bin/aws
./awscli-bundle/install -i /usr/local/aws -b /bin/aws

# Validate that the AWS CLI works
aws --version

# Clean up downloaded files
rm -rf /root/awscli-bundle /root/awscli-bundle.zip
```

#### 安装 openshift install 工具

```bash
# Get the OpenShift installer binary
OCP_VERSION=4.6.16
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-install-linux-${OCP_VERSION}.tar.gz
tar zxvf openshift-install-linux-${OCP_VERSION}.tar.gz -C /usr/bin
rm -f openshift-install-linux-${OCP_VERSION}.tar.gz /usr/bin/README.md
chmod +x /usr/bin/openshift-install

# Get the oc CLI tool
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux-${OCP_VERSION}.tar.gz
tar zxvf openshift-client-linux-${OCP_VERSION}.tar.gz -C /usr/bin
rm -f openshift-client-linux-${OCP_VERSION}.tar.gz /usr/bin/README.md
chmod +x /usr/bin/oc

# Check that the OpenShift installer and CLI are in /usr/bin
ls -l /usr/bin/{oc,openshift-install}

# 设置 oc 命令的自动补全
oc completion bash >/etc/bash_completion.d/openshift

```

 Ctrl+D 或者输入 exit 退出 root 用户.

#### 配置 aws 文件
保存 aws 认证文件到 $HOME/.aws/credentials
region 是可选性，可以选择离自己近的区域。要和之前创建环境时候选的region 一致，没测试过不一致是否会有问题。

Do NOT select us-east-1.
us-east-2 (Ohio)  
us-west-1 (California)  
eu-central-1 (Frankfurt)  
ap-southeast-1 (Singapore)  

把 AWSKEY 和 AWSSECRETKEY 换成实际信息
```bash
export AWSKEY=<YOURACCESSKEY>
export AWSSECRETKEY=<YOURSECRETKEY>
export REGION=us-east-2

mkdir $HOME/.aws
cat << EOF >>  $HOME/.aws/credentials
[default]
aws_access_key_id = ${AWSKEY}
aws_secret_access_key = ${AWSSECRETKEY}
region = $REGION
EOF
```

验证认证文件是否生效，正常会返回 Account， UserID，Arn

```bash
aws sts get-caller-identity

# 返回示例
{
    "Account": "657952645207",
    "UserId": "AIDAZSMIIUBLQD3YHF4Z6",
    "Arn": "arn:aws:iam::657952645207:user/wkulhane@redhat.com-b91e"
}
```

#### 配置 openshift install 文件

浏览器打开 https://www.openshift.com/try

选择 "In your public cloud" 的 “Try it in your cloud ->”
使用 Redhat 账户登录后，选择 “In the public cloud” 中的 AWS， -> Installer-Provisioned-Infrastructure

忽略其他参数，找到 pull secret，点 “Copy pull secret”，把内容保存到我们虚机上。

示例：

```bash
{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K3JocGRzYWRtaW5zcmVkaGF0Y29tMWZyZ3NpZHV6cTJkem5zajNpdzBhdG1samg3OjJMSTFEVTM1MFVCQks1ODRCTFVBODBFTTUABDDD13RDI0Qko2Q0I5VzNFSFIzS0pSSFhOSFgyVllNMlFFMVQ=","email":"youremail@redhat.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K3JocGRzYWRtaW5zcmVkaGF0Y29tMWZyZ3NpZHV6cTJkem5zajNpdzBhdG1samg3OjJMSTFEVTM1MFVCQks1ODRCTFVBODBFTTUABDDD13RDI0Qko2Q0I5VzNFSFIzS0pSSFhOSFgyVllNMlFFMVQ=","email":"youremail@redhat.com"},"registry.connect.redhat.com":{"auth":"NTE1NDg0ODB8dWhjLTFGckdzSURVWlEyRHpuU0ozaVcwQXRtTEpoNzpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSXpPR1prTVdJNFpqYzJOamcwTmpKbVlXTTRaVFpsWVRnd09EUTJOMkkzTnlKOS5YUmQ5LS1LQ3kzVlpVbF9ldTc0THpQMFEzOVYwRUVfeWRZOE5pVGRScUlyd2hVRHYtcFF2ZEtLV1ZpVmlaQWF0QkhEUVdmVDB1Z2pfTWIzYmNPUktqSXdBNldQTXYxWTc1RmhYQUg1S2Myc3lnSHVxWTRfZlhSOXJnbW42N0l0MmhiUXJyb3BBNXlaYXpXSzhPeTBJb29VWFAteDBPUjZ2VDJTVGktbm5sblBLbEFSWTBEZkxJYmk3OHZlZXFadUpyUDl4SzlXdnRaOEZOREpzQnlUc2VmeFRoVmtLMDVwVDlhTk9nTkxITGJMeU5sdEc1RE9xU1JiZ1hLMDJ6RXNaU3BwYmZLdVAwNVJYQWljQy14WEZiamtLaFpkYTgwV3lnZDJKcTZXWVF3WW83ZXgtLUh1MEpKeXBTczRINVY0Nm50dTNVRlNVUERBZEJ5VmVDU2RxckpzUWZoSmlpLVdJbXdjWnp6LUNwTlRfNVo0ei1WUkc0aV9hVF9TWnVkQzVySmFLdFpHS1RQWlg0SDlNLWxDeFlHZDJNYzhuWlc4NWVUeTJPYnBVOHA2S19sU3A3Wm15RzhEbWh6bFAtYTQzb0J1V3hJTHg3Y283U3BkOFRyYVNRbjVnaFpvc0VKZGp6X2ljTlFhVktNazFHQjEwbU1uOXJBeGdUcm5qU09aSEZvcXdmX2Y2dnZFWi0ySUp2Qk91UUZRQThsZDlzRDVDb1ZWNEdwTWx1Rl8zZGJqcXhuVTE0WXdHT2RhSldSOEtMTlFwbU9RV0JrWFJIcVpwN01UT0ZDX0dMVDRWeGNTMXhva0p6RUFxN1c4NzBSQVo4VnAtUGdscEJCc2RDT2tfdGNCNEY5T2hkZ0NPb3JMNHJkZmp6cEJobUZuMEhzVkFFNGJkaWhfRjNGSQ==","email":"youremail@redhat.com"},"registry.redhat.io":{"auth":"NTE1NDg0ODB8dWhjLTFGckdzSURVWlEyRHpuU0ozaVcwQXRtTEpoNzpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSXpPR1prTVdJNFpqYzJOamcwTmpKbVlXTTRaVFpsWVRnd09EUTJOMkkzTnlKOS5YUmQ5LS1LQ3kzVlpVbF9ldTc0THpQMFEzOVYwRUVfeWRZOE5pVGRScUlyd2hVRHYtcFF2ZEtLV1ZpVmlaQWF0QkhEUVdmVDB1Z2pfTWIzYmNPUktqSXdBNldQTXYxWTc1RmhYQUg1S2Myc3lnSHVxWTRfZlhSOXJnbW42N0l0MmhiUXJyb3BBNXlaYXpXSzhPeTBJb29VWFAteDBPUjZ2VDJTVGktbm5sblBLbEFSWTBEZkxJYmk3OHZlZXFadUpyUDl4SzszrefOEZOREpzQnlUc2VmeFRoVmtLMDVwVDlhTk9nTkxITGJMeU5sdEc1RE9xU1JiZ1hLMDJ6RXNaU3BwYmZLdVAwNVJYQWljQy14WEZiamtLaFpkYTgwV3lnZDJKcTZXWVF3WW83ZXgtLUh1MEpKeXBTczRINVY0Nm50dTNVRlNVUERBZEJ5VmVDU2RxckpzUWZoSmlpLVdJbXdjWnp6LUNwTlRfNVo0ei1WUkc0aV9hVF9TWnVkQzVySmFLdFpHS1RQWlg0SDlNLWxDeFlHZDJNYzhuWlc4NWVUeTJPYnBVOHA2S19sU3A3Wm15RzhEbWh6bFAtYTQzb0J1V3hJTHg3Y283U3BkOFRyYVNRbjVnaFpvc0VKZGp6X2ljTlFhVktNazFHQjEwbU1uOXJBeGdUcm5qU09aSEZvcXdmX2Y2dnZFWi0ySUp2Qk91UUZRQThsZDlzRDVDb1ZWNEdwTWx1Rl8zZGJqcXhuVTE0WXdHT2RhSldSOEtMTlFwbU9RV0JrWFJIcVpwN01UT0ZDX0dMVDRWeGNTMXhva0p6RUFxN1c4NzBSQVo4VnAtUGdscEJCc2RDT2tfdGNCNEY5T2hkZ0NPb3JMNHJkZmp6cEJobUZuMEhzVkFFNGJkaWhfRjNGSQ==","email":"youremail@redhat.com"}}}

```

检查文件中包含了这几个仓库 quay.io, registry.connect.redhat.com, registry.redhat.io, cloud.openshift.com

创建 ssh key 

```bash
ssh-keygen -f ~/.ssh/cluster-${GUID}-key -N ''
```

### 安装 openshift

执行下面的命令，相关需要键入的参数参照如下
```bash
openshift-install create cluster --dir $HOME/cluster-${GUID}

platform 选 aws
Region 自己选，不要选 us-east-1
Base Domain选 sandboxNNN.opentlc.com 
Cluster Name ： cluster-GUID，比如 cluster-6994
pull secret 将之前复制的内容贴进去，不要有空格，要连贯

# 等待过程中，可以敲敲回车，不然窗口长时间不动，会断开连接，安装也就中断了
```

可以再开一个窗口查看安装日志， tailf ${HOME}/cluster-${GUID}/.openshift_install.log

安装完成后，根据提示配置 KUBECONFIG 环境变量，就可以使用 oc 命令了。 
kubeadmin 的密码存在了文件 auth/kubeadmin-password
还有 web 控制台的地址，这些内容都保存下来。

```bash
INFO Creating infrastructure resources...
INFO Waiting up to 20m0s for the Kubernetes API at https://api.cluster-b91e.sandbox580.opentlc.com:6443...
INFO API v1.17.1 up
INFO Waiting up to 40m0s for bootstrapping to complete...
INFO Destroying the bootstrap resources...
INFO Waiting up to 30m0s for the cluster at https://api.cluster-b91e.sandbox580.opentlc.com:6443 to initialize...
INFO Waiting up to 10m0s for the openshift-console route to be created...
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/wkulhane-redhat.com/cluster-b91e/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.cluster-b91e.sandbox580.opentlc.com
INFO Login to the console with user: kubeadmin, password: GEveR-tBVTB-jJUJB-iC9Jn
```

可以把 KUBECONFIG 环境变量写入 profile，就不用每次手动敲 export
```bash
echo "export KUBECONFIG=/home/wkulhane-redhat.com/cluster-b91e/auth/kubeconfig" >> ~/.bash_profile
```

查看所有节点，状态都正常，安装完成。
```bash
[feng.cai-ibm.com@clientvm 0 ~]$ oc get node
NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-129-107.us-east-2.compute.internal   Ready    master   29m   v1.17.1
ip-10-0-135-99.us-east-2.compute.internal    Ready    worker   19m   v1.17.1
ip-10-0-153-175.us-east-2.compute.internal   Ready    worker   19m   v1.17.1
ip-10-0-156-115.us-east-2.compute.internal   Ready    master   29m   v1.17.1
ip-10-0-165-192.us-east-2.compute.internal   Ready    worker   19m   v1.17.1
ip-10-0-169-4.us-east-2.compute.internal     Ready    master   29m   v1.17.1
```

openshift-install create cluster 这条命令实现了自动化，也可以按照文档拆解为手动一步步的安装。
