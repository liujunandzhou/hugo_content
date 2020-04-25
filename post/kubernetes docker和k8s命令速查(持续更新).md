---
title: "kubernetes docker和k8s命令速查(持续更新)"
date: 2020-04-25T20:22:52+08:00
lastmod: 2020-04-25T20:22:52+08:00
draft: false
tags: ["k8s","docker","常见命令"]
categories: ["k8s","docker","常见命令"]
author: "铁血执着的青春"

---

# docker相关
  - docker ps -a
  - docker build -t xxx .  #根据Dockerfile生成image
  - docker images 
  - docker exec -it {ID\NAME} /bin/bash | /bin/sh
  - docker run -it {ID\NAME} -p -v 
  - docker start {ID\NAME}
  - docker stop {ID\NAME}
  - docker rm {ID\NAME}
  - docker rmi {ID\NAME}
  - docker save coredns/coredns:1.0.0 | gzip > coredns.tar.gz  #将已有的img打包
  - docker load -i IMAGE #docker载入本地image打包文件
  - docker stats {ID\NAME}  #查看容器资源占用情况，不填查看全部
  - docker cp 从本地拷贝内容到容器中
  - docker commit -p -m 'xxx' 1a889d5bbf99 xxx  #将容器打标签提交为img，-p选项意思是打包时暂停容器，不会退出
  - docker push xxx  #push到容器仓库


# kubernetes相关
  - kubectl get pods -o wide 
  - kubectl get pod xxx -o yaml  #获取yaml配置文件
  - kubectl get nodes -o wide 
  - kubectl set image deployment xxx xxx=image_url  #更改部署镜像
  - kubectl describe pod mysql-deploy-766bd7dbcb-7nxvw  #查看pod的创建运行记录
  - kubectl scale deploy/kube-dns --replicas=3    #修改deploy的副本数
  - kubectl create -f xxx.yaml  #创建资源
  - kubectl delete deploy mysql-deploy   #删除资源
  - kubectl get svc -o wide
  - kubectl get ep SVC_NAME   #查看svc对应绑定的pod
  - kubectl get rs
  - kubectl get deploy/DEPLOY-NAME
  - kubectl get all  #获取所有类型资源
  - kubectl get componentstatuses   #获取k8s各组件健康状态，简写为kubectl get cs
  - kubectl describe deploy/DEPLOY-NAME
  - kubectl status rollout deploy/DEPLOY-NAME
  - kubectl label nodes 171 disktype=ssd            #添加标签
  - kubectl label nodes 171 disktype-               #删除标签
  - kubectl label nodes 171 disktype=hdd --overwrite #修改标签
  - kubectl logs POD_NAME  #查看pod的日志，排错用
  - kubectl get nodes -o json | jq '.items[] | .spec'  #查看每个node的CIDR分配
  - kubectl delete pod NAME --grace-period=0 --force   #强制删除资源，在1.3版本去掉--force选项
  - kubectl replace -f xxx.yaml   #更改定义资源的yaml配置文件
  - kubectl get secret -n kube-system | grep dashboard  #查找secret
  - kubectl describe secret -n kube-system kubernetes-dashboard-token-ld92d  #查看该secret的令牌
  - kubectl scale --replicas=3 deployment/xxxx   #横向扩展deploy的rs数量
  - kubectl cordon NODENAME   #将node设置为检修状态，不再向此node调度新的pod
  - kubectl drain NODENAME    #将node设置为（排水）不可用状态，并且驱逐其上的pod转移至其他正常node上。这一步会进行两个步骤：1.将node设为cordon状态2.驱逐node上的pod
  - kubectl drain node2 --delete-local-data --force --ignore-daemonsets  #忽略ds，否则存在ds的话无法正常执行
  - kubectl uncordon NODENAME  #取消检修状态、排水状态
  - kubectl proxy --address=0.0.0.0 --port=8001 --accept-hosts=^.* &  #kubectl监听端口代理apiserver的服务，允许调用客户端免去认证步骤
  - kubectl patch deployment my-nginx --patch '{"spec": {"template": {"metadata": {"annotations": {"update-time": "2018-04-11 12:15" }}}}}'   #更新configmap之后挂载configmap 的pod中的环境变量不会自动更新，可以通过更新一下deployment.spec.template的注解内容来触发pod的滚动更新。