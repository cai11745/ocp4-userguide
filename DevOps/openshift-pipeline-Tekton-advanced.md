

使用的java应用源码
https://github.com/cai11745/spring-petclinic.git
在官方基础增加了 Dockerfile，k8s目录（deployment/service/route yaml），environments目录

- s2i-build-and-deploy.yaml openshift自带demo  

- s2i-build-and-deploy-mvn-mirror.yaml 增加了MAVEN_MIRROR_URL参数，可以使用从内网拉取编译所需文件，如 nexus  

- mvn-cache-and-deploy.yaml ，使用需要先导入 mvn-repo-task.yaml deploy-app-task.yaml 这两个task  
  支持MAVEN_MIRROR_URL，并在mvn package过程将repository缓存到存储，下次编译直接使用存储缓存文件，而不需要再次从仓库获取。 
  通过Dockerfile build image，Dockerfile默认要求放在git repo根目录，也可自定义路径，见 buildah 这个 clustertask  
  通过 deployment/service/route yaml 发布应用，yaml 文件放在 git repo 的 k8s目录，通过 environments/dev/kustomization.yaml 指定yaml路径  


pipelinerun-mvn-cache-and-deploy.yaml 快速根据mvn-cache-and-deploy这个pipeline生成一个pipelinerun，注意修改其中的pvc名称

oc get clustertasks.tekton.dev buildah -o yaml > buildah-cache-task.yaml

vim buildah-cache-task.yaml
kind: Task
  name: buildah-cache

  volumes:
  - name: varlibcontainers
    persistentVolumeClaim:
      claimName: var-lib-containers



### FAQ
####
STEP-BUILD

+ buildah --storage-driver=vfs bud --format=oci --tls-verify=true --no-cache -f ./Dockerfile -t image-registry.openshift-image-registry.svc:5000/demo-cicd/spring-petclinic:v1 .
chown /var/lib/containers/storage/vfs: operation not permitted
level=error msg="exit status 125"

sh-4.4# id
uid=0(root) gid=0(root) groups=0(root),1000670000

[root@bastion storageclass]# ls  demo-cicd-var-lib-containers-pvc-16b94c24-cace-45a8-be7a-285439fea406/storage/ -l
total 0
drwx------ 2 nfsnobody nfsnobody 6 May  7 23:18 mounts
-rw-r--r-- 1 nfsnobody nfsnobody 0 May  7 23:18 storage.lock
drwx------ 2 nfsnobody nfsnobody 6 May  7 23:18 tmp
-rw-r--r-- 1 nfsnobody nfsnobody 0 May  7 23:18 userns.lock
drwx------ 2 nfsnobody nfsnobody 6 May  7 23:18 vfs



[root@bastion demo-cicd-var-lib-containers-pvc-16b94c24-cace-45a8-be7a-285439fea406]# ls storage/ -l
total 0
drwx------ 2 root root  39 May  7 23:55 cache
drwx------ 2 root root   6 May  7 23:54 mounts
drwx------ 3 root root  15 May  7 23:54 overlay2
-rw-r--r-- 1 root root   0 May  7 23:54 storage.lock
drwx------ 2 root root   6 May  7 23:54 tmp
-rw-r--r-- 1 root root   0 May  7 23:54 userns.lock
drwx------ 3 root root  17 May  7 23:56 vfs
drwx------ 2 root root  29 May  7 23:55 vfs-containers
drwx------ 2 root root  25 May  7 23:55 vfs-images
drwx------ 2 root root 129 May  7 23:57 vfs-layers




### 5. 参考链接


https://ibm-developer.gitbook.io/cloudpakforapplications-appmod/ci-cd/tekton-tutorial-openshift

