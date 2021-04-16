



OpenShift Pipelines (Tekton) is GA'd from Openshift 4.4 onwards.


### 安装 Red Hat OpenShift Pipelines Operator (Tekton)

如果离线环境，需要先执行离线部署operatorhub  
https://github.com/cai11745/ocp4-userguide  可参照《离线部署operatorhub并批量导入》

1. 在控制台的 Administrator 视角中，Operators → OperatorHub。
2. 搜索 Red Hat OpenShift Pipelines Operator。点 Red Hat OpenShift Pipelines Operator 。
3. 在 Install Operator 页面中：
Installation Mode 选择 All namespaces on the cluster (default)。选择该项会将 Operator 安装至默认openshift-operators 命名空间，这将启用 Operator 以进行监视并在集群中的所有命名空间中可用。
Approval Strategy（批准策略）选择 Automatic。这样可确保以后对 Operator 的升级由 Operator Lifecycle Manager (OLM) 自动进行。

Update Channel：Stable 频道启用 Red Hat OpenShift Pipelines Operator 最新稳定版本的安装。preview 频道启用 Red Hat OpenShift Pipelines Operator 的最新预览版本，该版本可能包含 Stable 频道中还未提供的功能。

点击 Install。会看到 Installed Operators 页面中列出的 Operator。

检查 Status 变成 Succeeded 表示 Red Hat OpenShift Pipelines Operator 已安装成功。

```bash
查看相关api
[root@bastion ~]# oc api-resources --api-group=tekton.dev
NAME                SHORTNAMES   APIVERSION            NAMESPACED   KIND
clustertasks                     tekton.dev/v1beta1    false        ClusterTask
conditions                       tekton.dev/v1alpha1   true         Condition
pipelineresources                tekton.dev/v1alpha1   true         PipelineResource
pipelineruns        pr,prs       tekton.dev/v1beta1    true         PipelineRun
pipelines                        tekton.dev/v1beta1    true         Pipeline
runs                             tekton.dev/v1alpha1   true         Run
taskruns            tr,trs       tekton.dev/v1beta1    true         TaskRun
tasks                            tekton.dev/v1beta1    true         Task

相关pod  
[root@bastion ~]# oc get pod -n openshift-pipelines
NAME                                             READY   STATUS    RESTARTS   AGE
tekton-operator-proxy-webhook-5c86d47c54-kpcvx   1/1     Running   0          6h42m
tekton-pipelines-controller-78f7f7449d-wrcvb     1/1     Running   0          6h42m
tekton-pipelines-webhook-7885bc985b-54pc5        1/1     Running   0          7h10m
tekton-triggers-controller-76c8d6bd-kwgrk        1/1     Running   0          7h10m
tekton-triggers-webhook-6d6cfb6568-gl779         1/1     Running   0          7h10m
```

### demo 环境初始化安装
需要提前准备 storageclass 或者准备准备6个 5G的RWO空置pv，用户 gogs，nexus 等数据持久化。

```bash
oc new-project demo
git clone https://github.com/siamaksade/tekton-cd-demo
./demo.sh install
```

下一节会解读 demo.sh install 执行内容。

安装会从 quay.io 拉取镜像，过程可能会比较久，自行科学上网，或者用我阿里云地址。
quay.io/siamaksade/gogs:stable
->registry.cn-hangzhou.aliyuncs.com/laocai/gogs:stable  

quay.io/siamaksade/nexus3:3.16.2
->registry.cn-hangzhou.aliyuncs.com/laocai/nexus3:3.16.2

quay.io/chmouel/go-simple-uploader:latest
->registry.cn-hangzhou.aliyuncs.com/laocai/go-simple-uploader:latest

quay.io/siamaksade/nginx:latest
->registry.cn-hangzhou.aliyuncs.com/laocai/nginx:latest

quay.io/siamaksade/sonarqube:8.3-community
->registry.cn-hangzhou.aliyuncs.com/laocai/sonarqube:8.3-community 
上面几个镜像在 tekton-cd-demo/cd/*.yaml

quay.io/siamaksade/python-oc
->registry.cn-hangzhou.aliyuncs.com/laocai/python-oc:latest  
这个镜像在 tekton-cd-demo/config/gogs-init-taskrun.yaml  


脚本执行完成会提示 gogs，sonar，nexus 的访问地址。

```bash
  Demo is installed! Give it a few minutes to finish deployments and then:

  1) Go to spring-petclinic Git repository in Gogs:
     http://gogs-demo-cicd.apps.ocp4.example.com/gogs/spring-petclinic.git
  
  2) Log into Gogs with username/password: gogs/gogs
      
  3) Edit a file in the repository and commit to trigger the pipeline

  4) Check the pipeline run logs in Dev Console or Tekton CLI:
     
    $ tkn pipeline logs petclinic-deploy-dev -f -n demo-cicd

  You can find further details at:
  
  Gogs Git Server: http://gogs-demo-cicd.apps.ocp4.example.com/explore/repos
  Reports Server: http://reports-repo-demo-cicd.apps.ocp4.example.com
  SonarQube: https://sonarqube-demo-cicd.apps.ocp4.example.com
  Sonatype Nexus: http://nexus-demo-cicd.apps.ocp4.example.com
```

查看 demo-cicd project 下的pod ，pvc
```bash
[root@bastion tekton-cd-demo]# oc -n demo-cicd get pod
NAME                               READY   STATUS      RESTARTS   AGE
el-webhook-79f58545b9-c7pcw        1/1     Running     0          31m
gogs-646f4d9d95-fwsrj              1/1     Running     0          31m
gogs-postgresql-79877c948d-rf8kt   1/1     Running     0          31m
init-gogs-msjrj-pod-h7tml          0/1     Completed   0          9m38s
nexus-5d99746dc4-mwwxn             1/1     Running     0          31m
reports-repo-7c6b4f56d8-k68rd      2/2     Running     0          31m
sonarqube-6b8c69876c-2sfbr         1/1     Running     0          31m

[root@bastion tekton-cd-demo]# oc -n demo-cicd get pvc
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
gogs-data                   Bound    pvc-e419efa6-9882-466d-8e48-54124db51af1   1Gi        RWO            managed-nfs-storage   5m31s
gogs-postgres-data          Bound    pvc-b65639fa-e189-4946-b925-0e8d0911c8e7   1Gi        RWO            managed-nfs-storage   5m31s
nexus-pv                    Bound    pvc-661891f0-0d8f-403e-a101-5de7c259bfb7   5Gi        RWO            managed-nfs-storage   5m31s
petclinic-dev-workspace     Bound    pvc-4ad4a374-bfd7-46fa-bf56-9edd83ebb7a3   5Gi        RWO            managed-nfs-storage   5m29s
petclinic-stage-workspace   Bound    pvc-8497f2d4-f16a-42dc-8ef1-9c98a7606ec7   5Gi        RWO            managed-nfs-storage   5m29s
reports-repo-pv             Bound    pvc-90a8f378-67a9-499f-91c7-1b2382153d93   5Gi        RWO            managed-nfs-storage   5m31s
```

### demo.sh install 执行过程解析
对 demo.sh 内容进行查看，看下在 install 过程具体执行了哪些动作。

1. 创建 project
oc new-project demo-cicd
oc new-project demo-dev
oc new-project demo-stage

2. 对 service account pipeline 授权
oc policy add-role-to-user edit system:serviceaccount:demo-cicd:pipeline -n demo-dev
oc policy add-role-to-user edit system:serviceaccount:demo-cicd:pipeline -n demo-stage
oc policy add-role-to-user edit system:serviceaccount:demo-stage:default -n demo-dev

demo-cicd project 下的 pipeline sa 对 demo-dev 和 demo-stage project 具备 edit 权限。  
demo-stage project 下的 pipeline sa 对 demo-dev project 具备 edit 权限。

3. 在 demo-cicd project 部署 gogs，nexus，sonarqube

[root@bastion tekton-cd-demo]# ls cd/
gogs.yaml  nexus.yaml  reports-repo.yaml  sonarqube.yaml
[root@bastion tekton-cd-demo]# oc apply -f cd -n demo-cicd

此处会创建4个pvc用作数据持久化。

gogs 和 postgresql 用作 GitServer，存放代码  
nexus 
sonarqube 
reports-repo 

### Tekton 组件解读



参考内容
https://access.redhat.com/documentation/zh-cn/openshift_container_platform/4.7/html/cicd/installing-pipelines#op-installing-pipelines-operator-in-web-console_installing-pipelines

https://github.com/siamaksade/tekton-cd-demo



