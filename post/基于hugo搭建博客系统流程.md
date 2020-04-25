---
title: "基于hugo搭建博客系统流程"
date: 2020-04-25T20:28:52+08:00
lastmod: 2020-04-25T20:28:52+08:00
draft: false
tags: ["hugo"]
categories: ["hugo"]
author: "铁血执着的青春"

---

# 介绍

hugo是一款基于golang开发的静态站点生成工具,能够非常快速的基于markdown生成站点,并且有非常丰富主题可以供选择。它的主要特点:

* 部署简单。无需数据库等依赖,独立的二进制即可运行。
* 快速响应。基于markdown生成静态站点,服务器负载小。
* 操作简单。支持一键生成,以及自动生成。

所以对于很多需要搭建个人博客系统的人来说,hugo是不二选择。

# 安装

hugo的安装也比较简单。直接去官网下载对应二进制即可。

[官网包列表](https://github.com/gohugoio/hugo/releases)

可以点击如下链接,下载一个64位机的linux系统可以使用的版本。

[hugo_0.69.2_Linux-64bit.tar.gz](https://github.com/gohugoio/hugo/releases/download/v0.69.2/hugo_0.69.2_Linux-64bit.tar.gz)

获取到指定的二进制之后,就可以直接执行命令生成站点了。

# 运行

## 1. 生成我们需要的站点。
选任何一个目录,以我们的站点命名,生成一个静态网站的根目录。
```
hugo new site www.drivecode.cn
```
## 2. 安装指定theme。
```
cd www.drivecode.cn
git clone https://github.com/olOwOlo/hugo-theme-even themes/even
```
这样在根目录就会存在一个even目录了,目录下有一个exampleSite,里面有config.toml和content目录。

可以直接将config.toml和content目录,拷贝到项目根目录。

## 3. 运行hugo进程。
接下来就可以直接运行进程了,默认监听在1313端口。
```
/usr/local/hugo/bin/hugo server -p 1313 -v -w -D --appendPort=false -b http://www.drivecode.cn
```
本地目录下的变更,也能够实时感知到。

## 4. nginx配置转发。
因为云服务器上1313端口默认不对外开放,正常情况下,我们都是用80端口做一个二次转发。
于是我们就需要在nginx上做如下配置:

```
upstream hugo_web_page {
        server localhost:1313;
}

location / {
      proxy_pass http://hugo_web_page;
}

```

于是,通过[http://www.drivecode.cn](http://www.drivecode.cn)进行访问。

## 5. 配置systemd脚本。
我们可以配置一个hugo.service脚本,来保证hugo进程常驻内存。
```
[Unit]
Description=Hugo Project
After=network.service
 
[Service]
WorkingDirectory=/root/sites/www.drivecode.cn/
EnvironmentFile=-/usr/local/hugo/conf/conf
ExecStart=/usr/local/hugo/bin/hugo $HUGO_ARGS
Restart=always
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
```

其中的EnvironmentFile文件,就定义了具体的HUGO_ARGS配置文件。
/usr/local/hugo/conf/conf的具体内容如下:
```
HUGO_ARGS="server -p 1313 -v -w -D --appendPort=false -b http://www.drivecode.cn"
```

# 评论
## 集成评论
hugo可以集成的评论系统,有比较多的选择。这里主要是选择 **Valine** 这款评论系统作为评论的存储方案。

>Valine 是一款快速,简洁且高效的无后端评论系统,诞生于2017年8月7日,基于LeanCloud云服务平台来实现。

可以参考如下教程:
[https://valine.js.org/](https://valine.js.org/)

核心是在LeanCloud云平台上,创建一个应用,然后获取对应的: ${appId}和${appKey}.

然后在config.toml文件中配置如下:

```
  [params.valine]
    enable = true
    appId = '<appId>'
    appKey = '<appKey>'
    notify = false  # mail notifier , https://github.com/xCss/Valine/wiki
    verify = true # Verification code
    avatar = 'mp'
    placeholder = '说点什么吧...'
    visitor = true
```
至于此,评论系统就可以直接使用了。

## 删除评论
![
http://p.qpic.cn/qqconadmin/0/446d8aba6ba2409d9d09f82192b3d1dd/0](
http://p.qpic.cn/qqconadmin/0/446d8aba6ba2409d9d09f82192b3d1dd/0)

# 打赏
配置打赏功能,只需要获取微信打赏二维码和支付宝的二维码即可,配置如下:
```
  [params.reward]
    enable = true
    wechat = "<path-of-wx-qr-code-png>"   
    alipay = "<path-of-alipay-qr-code-png>"   
```

# 经验总结
1. hightlight的配置必须互斥。
>Syntax highlighting by Chroma. NOTE: Don't enable `highlightInClient` and `chroma` at the same time!

意思是: highlightInClient和chroma不能同时配置,否则可能出现页面展现不正常的问题。

# 参考文档

[知乎好用的hugo主题](https://www.zhihu.com/question/266175192?sort=created)

[好用的hugo主题列表](https://themes.gohugo.io/)

[Hugo 30min搭建静态博客网站](https://linux.cn/article-10048-1.html)

[valine一款快速简介评论系统]([https://valine.js.org/quickstart.html#%E8%8E%B7%E5%8F%96APP-ID-%E5%92%8C-APP-Key](https://valine.js.org/quickstart.html#获取APP-ID-和-APP-Key))

[leancloud serverless服务提供商,评论管理](https://leancloud.cn/dashboard/data.html?appid=cst7jFX7c5EREvLE2zJX1H2n-gzGzoHsz#/)

