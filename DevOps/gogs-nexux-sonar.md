





### demo 环境初始化安装
需要提前准备 storageclass 或者准备准备6个 5G的RWO空置pv，用于 gogs，nexus 等数据持久化。

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

mvn:3.5.0-jdk-8 -> 
registry.cn-hangzhou.aliyuncs.com/laocai/mvn:3.5.0-jdk-8
这个镜像在 tasks/mvn-task.yaml 

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
本节脚本不需要再执行。只为解读过程。  

cicd_prj dev_prj stage_prj 对应我们创建的三个demo project，可以在 demo.sh 中定义，此处分别对应 demo-cicd demo-dev demo-stage  

demo-cicd 运行公共组件 gogs、nexus等，及执行流水线
demo-dev 和 demo-stage 分别运行开发和模拟环境  

1. 创建 project
```bash
PRJ_PREFIX="demo"
dev_prj="$PRJ_PREFIX-dev"
stage_prj="$PRJ_PREFIX-stage"
cicd_prj="$PRJ_PREFIX-cicd"

oc new-project $cicd_prj 
oc new-project $dev_prj
oc new-project $stage_prj 
```

2. 对 service account pipeline 授权
```bash
oc policy add-role-to-user edit system:serviceaccount:$cicd_prj:pipeline -n $dev_prj
oc policy add-role-to-user edit system:serviceaccount:$cicd_prj:pipeline -n $stage_prj
oc policy add-role-to-user edit system:serviceaccount:$stage_prj:default -n $dev_prj
```

demo-cicd project 下的 pipeline sa 对 demo-dev 和 demo-stage project 具备 edit 权限，因为流水线是在 demo-cicd执行，需要对另外两个project 有读写 deployment 等资源的权限。  
demo-stage project 下的 pipeline sa 对 demo-dev project 具备 edit 权限。

3. 在 demo-cicd project 部署 gogs，nexus，sonarqube, reports-repo
```bash
[root@bastion tekton-cd-demo]# ls cd/
gogs.yaml  nexus.yaml  reports-repo.yaml  sonarqube.yaml
[root@bastion tekton-cd-demo]#   oc apply -f cd -n $cicd_prj

GOGS_HOSTNAME=$(oc get route gogs -o template --template='{{.spec.host}}' -n $cicd_prj)
```

此处会创建4个pvc用作数据持久化。

gogs 和 postgresql 用作 GitServer，存放代码  
nexus 用来缓存和存放第三方依赖库，编译后的制品
sonarqube 代码质量分析工具
reports-repo 一个upload服务配合nginx，用来存放和展示报告

4. 导入自定义task及pipeline

在 demo-cicd project导入自定义 task  
```bash
[root@bastion tekton-cd-demo]# ls tasks/
dependency-report-task.yaml  gatling-task.yaml  s2i-java-11-task.yaml
deploy-app-task.yaml         mvn-task.yaml

oc apply -f tasks -n $cicd_prj

# 导入 maven 配置，用于编译的时候将私服地址指向我们部署的nexus
oc create -f config/maven-settings-configmap.yaml -n $cicd_prj

# 创建两个 pvc， 用于 dev 和 stage 两个pipeline 的 workspace
oc apply -f config/pipeline-pvc.yaml -n $cicd_prj

# 将两个 pipeline 中的 project name。 git url替换成实际情况，也可以不改，创建pipelinerun 的时候修改  

sed -i "s/demo-dev/$dev_prj/g" pipelines/pipeline-deploy-dev.yaml 
sed -i "s#https://github.com/siamaksade#http://$GOGS_HOSTNAME/gogs#g" pipelines/pipeline-deploy-dev.yaml 

sed -i "s/demo-dev/$dev_prj/g" pipelines/pipeline-deploy-stage.yaml
sed -i "s/demo-stage/$stage_prj/g" pipelines/pipeline-deploy-stage.yaml
sed -i "s#https://github.com/siamaksade#http://$GOGS_HOSTNAME/gogs#g" pipelines/pipeline-deploy-stage.yaml

oc create -f pipelines -n $cicd_prj

# 导入 trigger
oc apply -f triggers/gogs-triggerbinding.yaml -n $cicd_prj
oc apply -f triggers/triggertemplate.yaml -n $cicd_prj
sed "s/demo-dev/$dev_prj/g" triggers/eventlistener.yaml | oc apply -f - -n $cicd_prj

```

5. 导入 gogs 配置并初始化

```bash
# 导入 gogs-config ，配置了 gogs 访问域名，访问postgres的地址及用户密码  
sed "s/@HOSTNAME/$GOGS_HOSTNAME/g" config/gogs-configmap.yaml | oc create -f - -n $cicd_prj
oc rollout status deployment/gogs -n $cicd_prj

# 执行了一个 taskrun，初始化 gogs 密码，及clone github demo 到 gogs  
oc create -f config/gogs-init-taskrun.yaml -n $cicd_prj
```

### 执行流水线





### 参考内容

https://github.com/siamaksade/tekton-cd-demo



