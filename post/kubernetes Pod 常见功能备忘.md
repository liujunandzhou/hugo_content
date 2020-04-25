---
title: "kubernetes Pod 常见功能备忘"
date: 2020-04-25T18:38:52+08:00
lastmod: 2020-04-25T21:41:52+08:00
menu: "main"
weight: 50

---

[TOC]
#### Pod声明周期和重启策略
Pod在整个生命周期中,被系统定义为各种状态。

| 状态值    | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| Pending   | API Server已经创建该Pod,但Pod内还有一个或多个容器的镜像没有创建成功,包括正在下载镜像的过程 |
| Running   | Pod内的所有容器均已创建,且至少有一个容器处于运行状态,正在启动,或正在重启状态 |
| Succeeded | Pod内所有的容器已成功执行退出,且不会在重启                   |
| Failed    | Pod内所有容器均已推出,但至少有一个容器退出为失败状态         |
| Unknown   | 由于某种原因,无法获取该Pod的状态,可能由于网络问题导致        |

##### Pod重启策略
Pod重启策略RestartPolicy应用于Pod内的所有容器，并且仅在Pod所处的Node上由kubelet进行判断和重启操作。当某个容器异常退出或者健康检查失败时，kubelet将根据RestartPolicy的设置来进行相应的操作。

Pod的重启策略包括Always、OnFailure和Never，默认值为Always
* Always：当容器失效时，由kubelet自动重启该容器
* OnFailure：当容器终止运行切退出代码不为0时，由kubelet自动重启该容器
* Never：不论容器运行状态如何，kubelet都不会重启该容器

kubelet重启失效容器的时间间隔以sync-frequency乘以2n来计算，例如1、2、4、8倍等，最长延时5min，并且在成功重启后的10min后重置该时间

Pod的重启策略与控制方式息息相关，当前可用于管理Pod的控制器包括ReplicationController,Job,DaemonSet及直接通过kubelet管理(静态Pod)。每种控制器对Pod的重启策略要求如下:

* RC和DaemonSet: 必须设置为Always,需要保证该容器持续运行.
* Job: OnFailure或Never,确保容器执行完成后不再重启.
* Kubelet: 在Pod失效时自动重启它,不论将RestartPolicy设置为什么值,也不会对Pod进行健康检查.

#### Pod健康检查
对于Pod的健康状态检查,可以通过两类探针来检查: **LivenessProbe**和**ReadinessProbe**
* LivenessProbe探针
用于判断容器是否存活(Running)状态,如果LivenessProbe探针检测到容器不健康,则kubelet将杀掉该容器,并根据容器的重启策略做相应的处理。如果一个容器不包含你LivenessProbe探针,那么kubelet认为该容器的LivenessProbe探针返回的值永远是"Success".

* ReadinessProbe探针
用于判断容器是否处于ready状态,并对外可以提供请求。如果ReadinessProbe探针检测到失败,则Pod的状态将被修改。Endpoint将从Service的EndPoint中删除包含该容器所在Pod的EndPoint,从而保证业务访问的Pod都是有效的。

##### 关键的探针配置参数说明

| 探针参数            | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| initialDelaySeconds | 容器启动完成和探针启动之间间隔的描述                         |
| periodSeconds       | 检查的频率(以秒为单位). 默认是10秒, 最小值为1                |
| timeoutSeconds      | 检查超时的秒数,默认值是1,最小值是1。可以根据场景设置。       |
| successThreshold    | 失败后探测成功的最小连续成功次数,默认为1..最小值为1。一般设置默认即可 |
| failureThreshold    | 当Pod成功启动且检查失败的时候,k8s放弃之前,将尝试failureThreshold次。放弃生存检查意味着重新启动Pod. 放弃就绪检查,将被标记为未就绪。默认值是3, 最小值未1. 一般设置默认值即可 |

通过表中的一些参数,可以算出服务从开始到可用的最小键值。也就是pod从创建到最终可用的时间间隔。

##### 探针的类型
* Http探针 发起要给Http请求。
* Exec 发起执行一个具体的命令。
* TCPSocket 表示检测一个TCP端口是否存活。