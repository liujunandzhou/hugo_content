---
title: "golang必知必会-浅谈golang依赖管理"
date: 2020-05-05T08:46:52+08:00
lastmod: 2020-05-05T08:46:52+08:00
draft: false
tags: ["golang","依赖管理"]
categories: ["golang","依赖管理"]
author: "铁血执着的青春"

---
> range应该是golang语言中,使用最为频繁的一个关键字了,稍有不慎,可能就导致bug。本文主要针对golang语言range的使用细节进行讲解。

## 为什么需要依赖管理?
软件开发过程中,为了提升开发效率,我们经常会引入很多第三方包。由于第三方包的更新,一般不受使用者的控制,因此有时候,会出现版本不兼容的问题。这是使用方就会出现代码无法编译通过的问题,或者由于特性变更,导致运行过程中不兼容的问题。同时在一个项目中,有可能同时依赖同一个库的不同版本,这个时候就需要依赖管理的问题了。

因此引入依赖管理的主要作用:
1. 更新
2. 版本控制
3. 多版本依赖问题.

## 依赖管理的方式
比较典型的集中方式如下:
1. GOPATH方式
2. vendor方式.
3. dep或者glide方式
4. go module方式.

### GOPATH方式
之前的做法是,给不同的项目设置不同的GOPATH,编译的时候实现动态切换. 每次编译一个项目,就切换到对应的GOPATH上,这样版本问题,被GOPATH隐藏了,但是使用删非常不方便。构建过程,需要切换到不同的分支.

### vendor方式
golang 1.5版本中,引入的一种机制,算是golang语言对版本管理的第一次支持。有了vendor机制,可以让依赖,跟着项目走。将项目的第三方依赖库,直接放置在工程根目录的vendorm目录下。
这样做有什么好处?
1. 如果你的项目是独立的项目,提交到git之后,别人下载到你的工程,就可以直接编译通过.
2. 如果你的项目是第三方库,那么使用你库的人,不会因为版本变更或者依赖问题,导致无法使用.

在低版本的golang中,使用vendor机制,需要手动开启。可以使用如下命令开启:
```
export GO15VENDOREXPERIMENT=1
```
[vendor说明](%28https://studygolang.com/articles/4607%29）)

主要有几个点:
1. vendor目录结构
2. vendor的嵌套和递归查找问题.

### dep或者glide工具
glide目前已经不维护了,原作者也是直接推荐使用官方的dep工具作为替代方案.

这里主要来说明下dep工具.
![
https://default-1256848756.cos.ap-guangzhou.myqcloud.com/dep%E5%B7%A5%E5%85%B7%E4%BD%BF%E7%94%A8.png](
https://default-1256848756.cos.ap-guangzhou.myqcloud.com/dep%E5%B7%A5%E5%85%B7%E4%BD%BF%E7%94%A8.png)

常用的命令主要是:
dep init
dep ensure
dep ensure -update
dep ensure -add "github.com/pkg/errors"

dep安装:
```
go get -u github.com/golang/dep/cmd/dep
```

执行dep init的时候,根目录下会多出三个文件:
```
Gopkg.toml
Gopkg.lock
vendor
```

这三者之间的关系如下:
![
https://default-1256848756.cos.ap-guangzhou.myqcloud.com/dep%E5%B7%A5%E5%85%B7.png](
https://default-1256848756.cos.ap-guangzhou.myqcloud.com/dep%E5%B7%A5%E5%85%B7.png)


dep对于golang.org等第三方网站下载支持不够好,需要业务手动处理下:
1)采用第三方的二次开发工具,针对GFW屏蔽的网站做了处理
```
https://github.com/cuigh/dep
```

2)去github.com/golang这个目录下下载然后替换.

### go module方式
这是一个在go1.11中提出的解决方案。使用方式上,通过GOPROXY环境变量指定仓库服务器的地方,对go client发出的命令进行响应,从而实现版本管理。
可以直接使用如下语句验证:
```
export GOPROXY=https://goproxy.io
```

或者使用Jfrog gocenter的地址:
```
export GOPROXY=https://gocenter.io
```

或者安装微软推出的一个goproxy工具,当然需要自行搭建使用.
 Athens – Go 依赖管理工具

 [Athens介绍](
https://mp.weixin.qq.com/s/NNUrVC6XFfUinshw5_4ehQ)

go module的特点:
1.搭建私服,类似于maven一样,公司拥有自己的仓库,便于管理,同时可以加速. 符合企业的要求.
2.go module支持replace功能,解决被墙的问题.
3.go module引入了语义规范,不同版本包名一样,必须提供兼容性.
4.go module支持多版本管理,非常好的支持依赖问题.