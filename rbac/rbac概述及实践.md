#### RBAC

RBAC(基于角色的访问控制)，一种基于用户的角色来管理对计算机或网络资源的访问的方法。

RBAC使用rbac.authorization.k8s.io API组进行授权决策，从1.8开始，RBAC模式使用稳定的rabc.authorization.k8s.io/v1API

要启用RBAC，需要在apiserver启动增加参数 " --authorization-mode=RBAC"

##### Role和ClusterRole

Role是一组权限的集合，权限没有"拒绝规则"，例如一个role可以包含读取Pod和列出Pod的权限；Role只能用户授权对单个命名空间内资源的访问权限。

例如，授予对默认命名空间下pod的读取权限

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]     # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```



ClusterRole功能与Role相同，但是它是在群集范围内进行授权。

例如，授予对所有命名空间中的secret的读访问权限

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

 ##### RoleBinding和ClusterRoleBinding

角色绑定将角色中定义的权限授予用户或用户组，授权对象(subject)可以是用户(User)，用户组(Groups)、用户账户(ServiceAccount)

RoleBinding在一个命名空间内授权，RoleBinding可以引用统一命名空间下的Role。例如：在命名空间"default"下，授予"pod-reader"角色给"jane"用户。

```
# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

RoleBinding也可以引用ClusterRole

```
# This role binding allows "dave" to read secrets in the "development" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets
  namespace: development # This only grants permissions within the "development" namespace.
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```



ClusterRoleBinding在集群范围和所有命名空间中授权，例如允许"manager"组中的所有用户读取任何命名空间下的secret

```
# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

##### kubernetes默认的RBAC

RBAC现在被 Kubernetes 深度集成，并使用他给系统组件进行授权。System Roles 一般具有前缀 system: 

`kubectl get clusterroles --namespace=kube-system `

#### 实例一 创建普通用户(ssl)

普通用户并不是通过k8s来创建和维护，是通过创建证书和切换上下文环境的方式来创建和切换用户。

##### 生成用户证书

创建私钥

`openssl genrsa -out custom.key 2048`

使用上述私钥创建证书签名请求custom.csr。-subj中指定了用户和用户组(CN用于用户名，O用户组)

`openssl req -new -key custom.key -out custom.csr -subj "/CN=custom/O=custom"`

找到kubernetes集群中证书颁发机构(CA)。它负责批准请求并生成访问集群API所需的证书(/etc/kuberntes/pki)

生成证书

`openssl x509 -req -in custom.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out custom.crt -days 1000`

##### 集群配置

设置集群参数

`kubectl --kubeconfig=config-cus config set-cluster custom --certificate-authority=/etc/kubernetes/pki/ca.crt --server=https://192.168.125.217:6443`

设置客户端认证参数

`kubectl --kubeconfig=config-cus config set-credentials jane --client-certificate=/root/config-exercise/openssl/custom.crt --client-key=/root/config-exercise/openssl/custom.key`

设置上下文参数

`kubectl --kubeconfig=config-cus config set-context custom-context --cluster=custom --user=jane`

至此，用户已经生成；但是创建的用户没有权限

##### 使用RBAC授予权限

授予用户custom在命名空间default下只读pod权限

创建role

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]     # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

创建rolebinding

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: User
  name: custom
  apiGroup: ""
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: ""
```

授予用户custom管理员权限(直接绑定系统自带的clusterrole cluster-admin)

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: exercise-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: User
  name: custom
  apiGroup: ""
```

##### 验证

查看上下文(不同上下文代表不同用户)

`kubectl --kubeconfig=config-cus config get-context`

切换上下文

`kubectl --kubeconfig=config-cus config use-context custom`

查看pod

`kubectl --kubeconfig=config-cus get po`

#### 实例二 创建普通用户(token)

##### 获取token

创建serviceaccount

`kubectl create servicaccount cce-service-account -n default`

获取serviceaccount下对应secret的token，并用base64解码

`token=$(kubectl get secret cce-service-account-token-g8dg5 -ncce -oyaml |grep token:| awk '{print $2}' | xargs echo -n | base64 -d)`

##### 创建用户

设置集群参数

`kubectl --kubeconfig=token-config config set-cluster cce-viewer --server=http://192.168.125.217:6443 --insecure-skip-tls-verify`

设置客户端token

`kubectl --kubeconfig=token-config config set-credentials cce-user --token=$token`

设置集群上下文

`kubectl --kubeconfig=token-config config set-context cce-viewer --user=cce-user  --cluster=cce-viewer`

##### 绑定RBAC

创建role，该权限为default密码空间pod只读

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

创建rolebinding

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: cce-service-account
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

