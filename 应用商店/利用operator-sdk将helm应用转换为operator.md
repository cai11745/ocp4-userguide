## 利用operator-sdk将helm应用转换为operator

### 简介
**Operator Framework**
Operator Framework 是 CoreOS 开源的一个用于快速开发 Operator 的工具包，该框架包含两个主要的部分：

Operator SDK: 无需了解复杂的 Kubernetes API 特性，即可让你根据你自己的专业知识构建一个 Operator 应用。
Operator Lifecycle Manager OLM: 帮助你安装、更新和管理跨集群的运行中的所有 Operator（以及他们的相关服务）

Operator SDK 配有一个 CLI 工具，可协助开发人员创建、构建和部署新的 Operator 项目。

开发 operator 有三种方式，GO，Ansible 和 helm
https://sdk.operatorframework.io/

### 基于 helm 创建 operator 
Operator 的主要功能是从代表应用程序实例的自定义对象中读取数据，并使其所需状态与正在运行的状态相匹配。对于基于 Helm 的 Operator，对象的 spec 字段是一个配置选项列表，通常在 Helm 的 values.yaml 文件中描述。

#### 安装 operator-sdk
https://sdk.operatorframework.io/docs/installation/install-operator-sdk/

可以获取二进制文件或者通过编译安装

```bash
// 通过获取二进制文件安装
// Set the release version variable
RELEASE_VERSION=v1.0.1

// Linux
curl -LO https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu
curl -LO https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/ansible-operator-${RELEASE_VERSION}-x86_64-linux-gnu
curl -LO https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/helm-operator-${RELEASE_VERSION}-x86_64-linux-gnu

chmod +x operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu && sudo mkdir -p /usr/local/bin/ && sudo cp operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu /usr/local/bin/operator-sdk && rm operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu

chmod +x ansible-operator-${RELEASE_VERSION}-x86_64-linux-gnu && sudo mkdir -p /usr/local/bin/ && sudo cp ansible-operator-${RELEASE_VERSION}-x86_64-linux-gnu /usr/local/bin/ansible-operator && rm ansible-operator-${RELEASE_VERSION}-x86_64-linux-gnu

chmod +x helm-operator-${RELEASE_VERSION}-x86_64-linux-gnu && sudo mkdir -p /usr/local/bin/ && sudo cp helm-operator-${RELEASE_VERSION}-x86_64-linux-gnu /usr/local/bin/helm-operator && rm helm-operator-${RELEASE_VERSION}-x86_64-linux-gnu

```

#### 创建 nginx operator demo （简单场景）
创建了一个 nginx-operator ，用于监听 APIVersion 为 example.com/v1apha1 、 Kind 为 Nginx 的资源对象。  

```bash
mkdir nginx-operator
cd nginx-operator
operator-sdk init --plugins=helm

operator-sdk create api --group demo --version v1alpha1 --kind Nginx

```

生成了以下文件
config  Dockerfile  helm-charts  Makefile  PROJECT  watches.yaml

helm-charts 里面就是 nginx 的 helm 文件  
watches.yaml 是监听的对象  
```bash
[root@k8s nginx-operator]# ls helm-charts/nginx/
charts  Chart.yaml  templates  values.yaml

[root@k8s nginx-operator]# cat watches.yaml 
# Use the 'create api' subcommand to add watches to this file.
- group: demo.my.domain
  version: v1alpha1
  kind: Nginx
  chart: helm-charts/nginx
# +kubebuilder:scaffold:watch
```

制作 operator 镜像，这个镜像是负责监听自定义资源 Kind： Nginx  
```bash
docker build -t registry.example.com:5000/nginx-operator:v1 .
docker push registry.example.com:5000/nginx-operator:v1

也可以使用这个命令，用法都在 Makefile 中定义了  
make docker-build docker-push IMG=registry.example.com:5000/nginx-operator:v1
```

创建CRD 以及 将上面的镜像部署出来  
```bash
make install
make deploy IMG=registry.example.com:5000/nginx-operator:v1

// 第一步是用 kustomize 工具把 config/crd 下的文件导入
[root@k8s nginx-operator]# cat Makefile |grep -A 2 ^install
install: kustomize
        $(KUSTOMIZE) build config/crd | kubectl apply -f -

// 将 default rbac manager crd 几个目录的内容导入
[root@k8s nginx-operator]# 
[root@k8s nginx-operator]# cat Makefile |grep -A 2 ^deploy
deploy: kustomize
        cd config/manager && $(KUSTOMIZE) edit set image controller=${IMG}
        $(KUSTOMIZE) build config/default | kubectl apply -f -


// pod 创建完成

[root@k8s nginx-operator]# kubectl -n nginx-operator-system get pod
NAME                                               READY   STATUS    RESTARTS   AGE
nginx-operator-controller-manager-9c6b5574-w6mcp   2/2     Running   0          8s

// 可以把 kustomize 工具放到 /usr/local/bin 下，省的后面每个 operator 都要下载一次  
[root@k8s nginx-operator]# cp bin/kustomize /usr/local/bin/
```

下面可以通过我们自定义的资源对象 kind: Nginx 发布应用  
提供了一个 demo，apiversion 和kind 就是要和初始化时候参数对应  
spec 的内容对应 helm 里的 value，nginx-operator-controller-manager 会将kind：Nginx 中的内容结合helm 转换成 k8s 的对象， deployment，service等，并持续监听这些资源的状态，若有不对会做修正。  
他也会去修改生成的nginx deployment，比如 一开始部署 replicaCount 是2 ，后面像改成3，要修改 kind： Nginx 里的参数，而不是直接去修改 nginx 的deployment。  

```bash
[root@k8s nginx-operator]# kubectl apply -f config/samples/demo_v1alpha1_nginx.yaml 

[root@k8s nginx-operator]# cat config/samples/demo_v1alpha1_nginx.yaml 
apiVersion: demo.my.domain/v1alpha1
kind: Nginx
metadata:
  name: nginx-sample
spec:
  # Default values copied from <project_dir>/helm-charts/nginx/values.yaml
  affinity: {}
  autoscaling:
    enabled: false
    maxReplicas: 100
    minReplicas: 1
    targetCPUUtilizationPercentage: 80
  fullnameOverride: ""
  image:
    pullPolicy: IfNotPresent
    repository: nginx
    tag: ""

...

```

部署完成，创建了 nginx-sample deployment 和 service  
```bash
[root@k8s nginx-operator]# kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
nginx-sample-646f977b4f-fsn97   1/1     Running   0          5m11s

[root@k8s nginx-operator]# kubectl get svc
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP   10h
nginx-sample   ClusterIP   10.96.202.136   <none>        80/TCP    5m36s
```

如果要修改内容，直接修改 nginx 这个对象，比如 replicaCount 改为2 , port 改为 8080

```bash
[root@k8s nginx-operator]# kubectl edit nginx nginx-sample

[root@k8s nginx-operator]# kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
nginx-sample-646f977b4f-fsn97   1/1     Running   0          9m34s
nginx-sample-646f977b4f-gs46g   1/1     Running   0          110s
[root@k8s nginx-operator]# kubectl get svc
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP    10h
nginx-sample   ClusterIP   10.96.202.136   <none>        8080/TCP   9m37s
```

如果要清除资源  
```bash
// 删除 nginx sample 
[root@k8s nginx-operator]# kubectl delete -f config/samples/demo_v1alpha1_nginx.yaml 

// 删除 nginx operator 及CRD
make undeploy

```

以上为通过一个很简单的 nginx demo 熟悉了通过 helm 来发布 operator 的过程。

#### 基于已有 helm 创建 operator
上述场景中，helm 都是 operator-sdk 负责创建完成的，若是我们已有 helm，如何转换成 operator  
找一个稍微复杂一些的有状态应用做测试，以 elasticsearch 为例

https://sdk.operatorframework.io/docs/building-operators/helm/migration/
https://sdk.operatorframework.io/docs/building-operators/helm/tutorial/

**安装 helm3**

```bash
 curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
 chmod 700 get_helm.sh
 ./get_helm.sh 
 
 // 添加 常用 repo
 helm repo add stable https://kubernetes-charts.storage.googleapis.com
 helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
 helm repo add ibmstable https://raw.githubusercontent.com/IBM/charts/master/repo/stable
 helm repo add bitnami https://charts.bitnami.com/bitnami
```

查找下 elasticsearch ， stable 里的已经弃用，使用 bitnami/elasticsearch 
```bash
[root@k8s ~]# helm search repo elasticsearch
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
bitnami/elasticsearch           12.6.4          7.9.1           A highly scalable open-source full-text search ...
incubator/elasticsearch         1.10.2          6.4.2           DEPRECATED Flexible and powerful open source, d...
incubator/elasticsearch-curator 0.4.3           5.5.4           A Helm chart for Elasticsearch Curator            
incubator/fluentd-elasticsearch 2.0.7           2.3.2           DEPRECATED! - A Fluentd Helm chart for Kubernet...
stable/elasticsearch            1.32.5          6.8.6           DEPRECATED Flexible and powerful open source, d...
stable/elasticsearch-curator    2.2.1           5.7.6           A Helm chart for Elasticsearch Curator            
stable/elasticsearch-exporter   3.7.0           1.1.0           Elasticsearch stats exporter for Prometheus   
...

```

获取 es chart 文件，并通过 operator-sdk 命令将他初始化成为一个 operator 的对象  
```bash
mkdir elasticsearch
cd elasticsearch
helm pull bitnami/elasticsearch

operator-sdk init --plugins=helm --domain=example.com

operator-sdk create api \
  --group=cache \
  --version=v1 \
  --kind=Elasticsearch \
  --helm-chart=elasticsearch-12.6.4.tgz

// --helm-chart 参数可以指定 chart 压缩文件，也可以是解压后目录  
// 命令完成，创建了一个新的api 是 cache.example.com/v1 ， 资源对象是 Elasticsearch
// 并将 helm chart 的内容copy 到了当前的 helm-charts 目录
// helm-charts 其中helm内容可以按照我们的需要进行修改

```

制作operator管理镜像，并部署  
```bash
make docker-build docker-push IMG=registry.example.com:5000/elasticsearch-operator:v1

make install
make deploy IMG=registry.example.com:5000/elasticsearch-operator:v1

// 部署完成
[root@k8s elasticsearch]# kubectl -n elasticsearch-system get pod
NAME                                                READY   STATUS    RESTARTS   AGE
elasticsearch-controller-manager-79c6dc7dd7-4gn4k   2/2     Running   0          38s
```









参考文档：
https://sdk.operatorframework.io/docs/building-operators/helm/quickstart/#