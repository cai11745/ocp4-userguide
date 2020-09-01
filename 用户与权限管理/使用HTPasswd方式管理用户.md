## 使用 HTPasswd 方式管理用户

当我们用默认的 kubadmin 进入平台的时候，会有一行提示信息，告诉我们可以配置用户管理，来允许其他用户登入。   
这也是一个平台使用过程中的必备步骤，openshift 支持 htpasswd, ldap, github, google 等用户系统对接。  
最常用的还是 htpasswd 和 ldap

本节介绍 htpasswd 方式配置用户登录，此方式用户和密码将由 openshift 自身来管理。 与ocp v3 的差异有两点
1. 支持页面导入了  
2. 密码文件不再是以文件方式保存在虚机上，而是以 secret 保存 ocp etcd 中

登录的提示信息  
` You are logged in as a temporary administrative user. Update the cluster OAuth configuration to allow others to log in. `

### 1. 使用 oc 命令管理 htpasswd 作为 ocp 用户管理

#### 1.1 首先创建密码文件

第一个用户参数需要带 -c 创建文件，后面添加用户不要带c，不然会覆盖文件。

```bash
# 生成一个 htpasswd.txt 文件，用户名user1，密码：123456
# 并添加两个用户 user1 user2 ,完成后看下文件，确认有这三个用户
htpasswd -c -B -b htpasswd.txt user1 123456
htpasswd -B -b htpasswd.txt user2 123456
htpasswd -B -b htpasswd.txt user3 123456
```

#### 1.2 创建密钥导入上述 htpasswd 密码文件
把密码文件导入到 openshift-config project 中，secret 名称 htpass-secret 可以自定义， --from-file 后面的 htpasswd 必须是这个字段，不能修改。

```bash
oc create secret generic htpass-secret --from-file=htpasswd=htpasswd.txt -n openshift-config

[root@bastion install]# oc get secret -n openshift-config htpass-secret
NAME            TYPE     DATA   AGE
htpass-secret   Opaque   1      25s
```

#### 1.3 创建 oauth CR
创建一个 oauth  Custom Resource (CR) ，使用上面的 htpasswd

```bash
cat oauth-htpasswd.yaml

apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: my_htpasswd_provider  #1 
    mappingMethod: claim 
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret  #2
```

1. 这是显示在登录页面的字段
2. 这个要和上一步得 secret 名称匹配

导入文件，可以会有个 warning，可忽略

```bash
oc apply -f oauth-htpasswd.yaml

# 忽略 warning
Warning: oc apply should be used on resources created by either oc create --save-config or oc apply.

```

#### 1.4 测试新用户登录
通过命令或者 web 页面登录测试新用户。 这些用户进去之后都只能自己新建 projecct 操作，没有赋权全局。

```bash
oc login -u user1
或
oc login https://api.ocp4.example.com:6443
```

这边 oc 命令登录出现了一些问题，页面是可以的，oc 命令不行。  

oc login error: x509: certificate signed by unknown authority  
解决方法: 
mv ~/.kube/config /tmp/

Login failed (401 Unauthorized)  
解决方法：
这个应该 user1 用户之前登录过，参照下一步的特别注意，清除之前的 user1 用户。注意别删错了，secret 不能删。

#### 1.5 增删用户
如果要增删用户，修改之前创建的 secret

```bash
# 先把 secret 导出来，转成明文
oc get secret htpass-secret -ojsonpath={.data.htpasswd} -n openshift-config | base64 -d > users.htpasswd

# 针对密码文件增删用户
# 增加用户
htpasswd -bB users.htpasswd <username> <password>
# 删除已有用户
htpasswd -D users.htpasswd <username>

# 覆盖之前的 secret
oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd --dry-run -o yaml -n openshift-config | oc replace -f -
```

**特别注意**
如果移除了用户，以下两处已存在的资源也需要删除。如果修改了 identityProviders，这两处的用户信息也需要删除，不然登录会出错

1. 删除 user
` oc delete user <username>  `
不删除 user 的话，后续这个用户还可以登录。

2. 删除  identity 中的 user
` oc delete identity my_htpasswd_provider:<username> `

#### 1.6 用户权限设置
给 user1 集群管理员权限，这是最大权限  
给 user2 cluster-reader 权限，可以查看所有 project 下的资源
给 user3 赋予 openshift-monitoring project 管理员权限

需要user2， user3 先登录过一次，不然 ocp 的 user 资源对象里找不到他，加不上权限

```bash
oc adm policy add-cluster-role-to-user cluster-admin user1
oc adm policy add-cluster-role-to-user cluster-reader user2
oc adm policy add-role-to-user admin user3 -n openshift-monitoring
```

### 2. 通过 web 页面配置 oauth 采用 htpasswd 认证
除了通过命令导入也可以通过页面，前提是先创建好 htpasswd 文件，等于说 secret 和 oauth 的资源是web 页面帮助创建的。  

通过上面的提示信息跳转，或者 Administrator 角色，进入菜单 Administration--Cluster Settings--Global Configuration--Oauth  

在 Identity Providers 的 Add 下拉菜单找到 HTPasswd

进入配置页面，name自定义，可不改。  
HTPasswd File 上传我们刚刚生成的 htpasswd.txt 文件，完成后点击 Add 即完成。
