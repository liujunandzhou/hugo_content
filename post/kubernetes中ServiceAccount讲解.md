---
title: "kubernetes中ServiceAccount讲解"
date: 2020-04-25T19:30:52+08:00
lastmod: 2020-04-25T19:30:52+08:00
draft: false
tags: ["k8s","docker","ServiceAccount"]
categories: ["k8s","docker","ServiceAccount"]
author: "铁血执着的青春"

---

# ServiceAccount是什么
## 账号的分类
所有的kubernetes集群中账户分为两类,Kubernetes管理的**ServiceAccount(服务账户)** 和 **UserAccount(用户账户)**.
* **UserAccount**主要是用于用户通过终端命令(kubecel)来管理整个集群。
* **ServiceAccount**主要是用于Pod中的进程和外部用户提供身份信息。并基于http token的方式来实现身份验证。

## 两种账号存在的区别
* 面向的对象不同。UserAccount主要是用来给人使用的,ServiceAccount主要是给进程使用的。
* 作用范围不同。UserAccount主要是作用与全局的,但是ServiceAccount主要是作用于某一个namespace。
* 复杂度不同。UserAccount创建比较复杂。但是ServiceAccount创建比较轻量级,创建容易,比较好进行控制。

这里讲解了账号类型,也大概说一下具体的权限管理过程。
在kubernetes中,一个请求会经过认证和授权两个阶段。
* **认证阶段**: 主要是验证用户的合法性。
* **授权阶段**: 主要是验证用户是否有权限操作。

## 认证的三种方法
1. bearer token方式。 采用jwt的方案,通过token来验证用户账号的合法性。
2. 客户端证书方式。目前比较主流的方案, 通过客户端证书认证用户的合法性。
3. HTTP BASE方式。通过账号和密码认证,在http头部附加Authentication头部校验。

认证通过之后,就可以捕获到用户的账号和所属的group信息。接下来就可以进行授权校验了。
授权的基本过程如下:
![
http://p.qpic.cn/qqconadmin/0/5a63235e23a849eb98cf7d359ef0ee1a/0](
http://p.qpic.cn/qqconadmin/0/5a63235e23a849eb98cf7d359ef0ee1a/0)

通过配置role和用户的关系,从而实现授权操作。这里的用户既可以是UserAccount,也可以是ServiceAccount。只是两者解决的场景不同而已。所以单纯的ServiceAccount其实是没有什么作用的,核心还是在于与Role和ClusterRole的绑定。

# ServiceAccount详解
## ServiceAccount流程图
网上这张图非常形象的说明了具体的过程:
![
http://p.qpic.cn/qqconadmin/0/b6febce1635a4140b5b10e25f247da43/0](
http://p.qpic.cn/qqconadmin/0/b6febce1635a4140b5b10e25f247da43/0)


## ServiceAccount的特点
1. 每个namespace下面都有一个名为default的ServiceAccount. 并且有一个与之对应的叫做default-token的secret对象。在不指定ServiceAccount的情况下,默认的secret会进行挂载,从而实现认证过程。
2. 每创建一个ServiceAccount对象,都会默认生成一个与之对应的叫做token的secret对象。
3. ServiceAccount创建之后,可以在Pod中通过**serviceAccountName**指定。
4. ServiceAccount创建之后,与**Role或者CluserRole**绑定,从而实现Pod获取不同的权限。

## ServiceAccount的原理
我们都知道ServiceAccount是基于http token的方式来的。那么这个token又是怎么生成的呢? 

### 必须理解的三个文件
* /var/secrets/kubernetes.io/serviceaccount/token
* /var/secrets/kubernetes.io/serviceaccount/ca.crt
* /var/secrets/kubernetes.io/serviceaccount/namespace

### 验证详细流程
1. Token的内容来自于Pod里指定路径下的一个文件(/var/secrets/kubernetes.io/serviceaccount/token),这种token是动态生成的,确切的说,是由kubernetes controller进行用API Server的私钥(--service-account-private-key-file指定的私钥)签名生成的一种JWT Secret。
2. 官方提供的客户端REST框架代码里,通过HTTPS的方式与API SERVER建立连接后,会用POD里指定路径下的一个CA证书(/var/secrets/kubernetes.io/serviceaccount/ca.crt)，验证API Server发送过来的证书,验证发送过来的证书是否是CA签发的合法证书。
3. API Server收到这个token之后,会采用自己的私钥(实际的参数是--service-account-private-key-file指定的私钥,如果没有设置此参数,那么就使用--tls-private-key-file指定的参数)来对Token的合法性进行验证。

所以通过上面的讲解,我们可以得出三个文件的作用:
* /var/secrets/kubernetes.io/serviceaccount/token
用来真正实现与API Server通信的认证token,是JWT格式的。可以通过命令查看。

* /var/secrets/kubernetes.io/serviceaccount/ca.crt
是服务端配置的CA证书,主要是用来实现安全https通信校验的。

* /var/secrets/kubernetes.io/serviceaccount/namespace
主要是用来指定一个具体的namespace的。因为ServiceAccount是隶属于namespace的。会作为请求,一并发送到Api Server端。

### ServiceAccount和Secret的关系
就像上面的讲解的ServiceAccount和Sercrets的关系: 默认创建好一个sa之后,就会创建一个对应的secrets,并且secrets中有三个子项: token,ca.crt,namespace.

通过如下命令可以清晰的看到:
```
kubectl describe serviceaccounts
```
查看serviceaccounts的构成:
![
http://p.qpic.cn/qqconadmin/0/ad09f66675bf460faaccfb30e7bb7cff/0](
http://p.qpic.cn/qqconadmin/0/ad09f66675bf460faaccfb30e7bb7cff/0)

通过如下命令,可以查看secrets的构成:
```
kubectl describe secrets default-token-ksm2b
```
注意: default-token-ksm2b是根据上面的输出选择的。不同用户的输出不一样。
输出如下结果:
![
http://p.qpic.cn/qqconadmin/0/af60f62c711b406191a4628761a4016d/0](
http://p.qpic.cn/qqconadmin/0/af60f62c711b406191a4628761a4016d/0)

从上面可以看出: 每一个ServiceAccount都会对应一个类型为kubernetes.io/service-account-token的Secret. 并且在pod创建的时候,挂在到pod的/var/secrets/kubernetes.io/serviceaccount目录下。


### ServiceAccount和Sercret的运行机制
在前面的Controller manager原理分析中,我们知道Controller中有两个非常特殊的控制器,分别是: **ServiceAccount Controller和Token Controller.**

* 其中ServiceAccount Controller监听者ServiceAccount和Namespace的事件。如果一个namespace没有default service account, 那么ServiceAccount Controller就会给它创建一个默认的ServiceAccount。
* 如果Controller manager在进程启动时,指定了API Server的密钥(通过参数--service-account-private-key-file),那么这个时候,就会启动Token Controller. Token Controller也会监听Service Account事件,如果发现新创建的Service Account没有对应的Service Account Secret,那么就会使用API Server配置的私钥指定一个Token(JWT TOKEN),并用该Token,CA证书以及namespace名称生成一个新的ServiceAccount Secret. 然后放入Service Account中。
* 当我们在API Server的鉴权过程中,启用了Service Account类型的准入控制器。即在kube-apiserver启动参数中,包括如下的内容:
```
--admission_control=ServiceAccount
```
那么针对Pod的新增或者删除请求,Service Account就会验证Pod里面指定的Service Account是否合法。

具体的过程如下:
1. 如果spec.ServiceAccount域没有被设置,则Kubernetes默认为其指定名字为default的Service Account.
2. 如果Pod的spec.ServiceAccount指定了default之外的ServiceAccount，并且该ServiceAccount没有被创建,则该Pod操作失败。
3. 如果在Pod中没有指定"ImagePullSecrets",那么这个spec.serviceAccount中对应的ServiceAccount的"ImagePullSecrets"会被加入到Pod中。
4. 给Pod添加一个特殊的Volumn,在该Volumn中包含Service Account Secret中的token信息,并将该Volumn挂在到/var/secrets/kubernetes.io/serviceaccount目录下。

所以说: ServiceAccount能够正常工作,离不开如下三个控制器.
1. Admission Controller.
2. Token Controller.
3. Service Account Controller.

# ServiceAccount操作
## ServiceAccount查看
```
kubectl get sa
```
![
http://p.qpic.cn/qqconadmin/0/eca92268032844f2b43bd9267f894882/0](
http://p.qpic.cn/qqconadmin/0/eca92268032844f2b43bd9267f894882/0)

## ServiceAccount对应Secret查看
```
kubectl get secrets
```
![
http://p.qpic.cn/qqconadmin/0/2fe9d067ed704c5c925578a661890982/0](
http://p.qpic.cn/qqconadmin/0/2fe9d067ed704c5c925578a661890982/0)

## SeviceAccount创建
```
kubectl create serviceaccount manager
```

## ServiceAccount与Pod绑定
```
apiVersion: v1
kind: Pod
metadata:
  name: my-sa-demo
  namespace: default
  labels:
    name: myapp
    tier: appfront
spec:
  containers:
  - name: myapp
    image: nginx
    ports:
    - name: http
      containerPort: 80
  serviceAccountName: manager

```

# 参考文档
[kubernetes之ServiceAccount](https://www.cnblogs.com/xzkzzz/p/9889173.html)
[RBAC权限控制](https://blog.csdn.net/ywq935/article/details/84840935)