## 移除 kubeadmin -- 可选项。高风险，谨慎操作

当已经添加一个新的 identity provider，并对至少一个新用户赋权了 cluster-admin  

可参照上一节的  [使用HTPasswd方式管理用户](./使用HTPasswd方式管理用户.md)

系统安装完自带的管理员 kubeadm 可以移除，可选操作，非必备。移除后登录页面就不会出现 kube:admin 这个选项了 。

**如果还没有赋权 cluster-admin 用户，先把 kubeadmin 给删了，你的ocp 集群就只能重装了，不可逆操作。**

移除前需确认以下内容，不然删完系统就崩了：

1. 至少已配置一个可用的 identity provider 
2. 已经对一个新用户赋予了 cluster-admin 权限
3. 使用了管理员登录，就是上面具备 cluster-admin 的用户

```bash
# 先使用自带的管理员给新用户授权 cluster-admin
oc login https://api.ocp4.example.com:6443 -u kubeadmin
oc adm policy add-cluster-role-to-user cluster-admin user1

# 使用新用户登录
oc login https://api.ocp4.example.com:6443 -u user1

# 确认 user1 权限，必备！！！
## 能看到所有project，应该就不会有问题
oc projects 

## 也可以再深一步确认。 看最新的 clusterrilebinds， 就是刚刚我们 oc admin policy 命令生成的  
oc get clusterrolebindings --sort-by=.metadata.creationTimestamp
oc get clusterrolebindings cluster-admin-0 -o yaml

clusterrolebinding 是把一个角色比如 cluster-admin 赋予一个用户比如 user1 或者 service accccount

内容如下：
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding  #表示这是一个全局的权限绑定， 对所有project 生效
metadata:
  creationTimestamp: "2020-06-25T10:05:09Z"
  name: cluster-admin-0
  resourceVersion: "1649419"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/cluster-admin-0
  uid: 05862866-cfa5-4872-9ab5-bc266e823b63
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin   #角色，集群管理员
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: user1  #我们的用户


# 现在可以删除 kubeadmin 了
oc delete secrets kubeadmin -n kube-system
```

现在 web 页面登录集群，kube:admin 那个窗口不见了。仅有的 htpasswd 变成了集群默认的认证方式。