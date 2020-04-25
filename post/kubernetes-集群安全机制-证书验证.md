---
title: "kubernetes 集群安全机制-证书验证"
date: 2020-04-25T19:57:52+08:00
lastmod: 2020-04-25T19:57:52+08:00
draft: false
tags: ["k8s","docker","证书"]
categories: ["k8s","docker","证书"]
author: "铁血执着的青春"

---

# 说明
这篇文章是为了完整的记录一下集群安全机制的实操过程。并对操作过程中碰到的问题进行一下总结和回顾。

# 计划
1. 创建一个admin用户,用来管理集群。
2. 创建一个pod-reader用户,用来管理pod配置。
3. 对admin用户绑定到默认的cluster-admin这个ClusterRole.
4. 对于pod-reader,绑定一个用户自定义的Role.

# 认证
## 生成admin证书
1. 生成admin的csr配置文件
```
cfssl print-defaults csr > client-admin-csr.json
```

2. 修改配置文件
将CN修改为admin,将O配置为admin
```
{
    "CN": "admin",
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "O": "admin",
            "C": "US",
            "L": "CA",
            "ST": "San Francisco"
        }
    ]
}
```
注意上图中的CN和O选项。

3. 生成证书配置.
```
cfssl gencert -ca=../apiserver/ca.pem -ca-key=../apiserver/ca-key.pem -config=../apiserver/ca-config.json -profile=client  client-admin-csr.json | cfssljson -bare client-admin
```

执行之后就会生成如下三个文件:
* 证书csr文件: client-admin.csr
* 证书私钥文件: client-admin-key.pem
* 证书公钥文件: client-admin.pem

## 生成pod-reader证书
1. 生成pod-reader的csr配置文件
```
cfssl print-defaults csr > client-pod-reader-csr.json
```

2. 修改配置文件
将CN修改为admin,将O配置为admin
```
{
    "CN": "pod-reader",
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "O": "pods",
            "C": "US",
            "L": "CA",
            "ST": "San Francisco"
        }
    ]
}
```
注意上图中的CN和O选项。

3. 生成证书配置.
```
cfssl gencert -ca=../apiserver/ca.pem -ca-key=../apiserver/ca-key.pem -config=../apiserver/ca-config.json -profile=client  client-pod-reader-csr.json | cfssljson -bare client-pod-reader
```

执行之后就会生成如下三个文件:
* 证书csr文件: client-pod-reader.csr
* 证书私钥文件: client-pod-reader-key.pem
* 证书公钥文件: client-pod-reader.pem


## 配置kubeconfig配置
1. 创建kubernetes集群。
```
kubectl config set-cluster kubernetes \
  --certificate-authority=/root/k8s-dockerfile/ssl/apiserver/ca.pem \
  --embed-certs=true \
  --server=https://192.168.122.1:6443
```

2. 创建pod-reader用户。
```
kubectl config set-credentials pod-reader \
  --client-certificate=/root/k8s-dockerfile/ssl/client/client-pod-reader.pem \
  --embed-certs=true \
  --client-key=/root/k8s-dockerfile/ssl/client/client-pod-reader-key.pem
```

3. 创建admin用户。
```
kubectl config set-credentials admin \
  --client-certificate=/root/k8s-dockerfile/ssl/client/client-admin.pem \
  --embed-certs=true \
  --client-key=/root/k8s-dockerfile/ssl/client/client-admin-key.pem
```

4. 创建两个context.
创建admin-k8s上下文:
```
kubectl config set-context admin-k8s \
  --cluster=kubernetes \
  --user=admin
```

创建pod-reader-k8s上下文:
```
kubectl config set-context pod-reader-k8s \
  --cluster=kubernetes \
  --user=pod-reader
```
5. 指定context.
```
kubectl config use-context admin-k8s
```

# 授权
## 绑定admin用户到cluster-admin到clusterRole
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin 
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: admin
```

其中cluster-admin是系统自定义的ClusterRole,类似于管理员的角色。
查看当前系统定义的clusterRole可以使用:
```
kubectl get clusterrole
```
输出信息如下图:

![
http://p.qpic.cn/qqconadmin/0/bf427f584131431384919b3a73437319/0](
http://p.qpic.cn/qqconadmin/0/bf427f584131431384919b3a73437319/0)

## 创建pod-reader这个Role
创建一个pod-reader的Role,对namespace为default,具有获取和查看权限。
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default 
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
  
```
## 绑定pod-reader用户到pod-Reader role
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader 
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: pod-reader
```

## 权限功能验证
1. 切换到admin这个context.
```
kubectl config use-context  admin-k8s
kubect get pods
```
输出结果:
![
http://p.qpic.cn/qqconadmin/0/34e8f75744c444d6ae7e953b73a99f51/0](
http://p.qpic.cn/qqconadmin/0/34e8f75744c444d6ae7e953b73a99f51/0)

2.切换到pod-reader这个context
```
kubectl config use-context  pod-reader-k8s
kubectl get pods
kubectl get nodes
```
输出结果如下:
* 查看pods列表
![
http://p.qpic.cn/qqconadmin/0/34e8f75744c444d6ae7e953b73a99f51/0](
http://p.qpic.cn/qqconadmin/0/34e8f75744c444d6ae7e953b73a99f51/0)
* 查看nodes结果
![
http://p.qpic.cn/qqconadmin/0/7fe695b9616e452db5920d0b12c93654/0](
http://p.qpic.cn/qqconadmin/0/7fe695b9616e452db5920d0b12c93654/0)

可以非常明显的看到,对于pods是list权限。但是对于nodes,没有list权限。

# 参考文档
[k8s认证授权操作记录](
https://blog.csdn.net/IT8421/article/details/89389609)

[使用token或者证书认证](
https://www.dazhuanlan.com/2019/11/30/5de162f47e8cf/)