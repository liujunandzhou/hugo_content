---
title: "golang xx2togo的工具使用"
date: 2020-05-03T23:24:52+08:00
lastmod: 2020-05-03T23:24:52+08:00
draft: false
tags: ["golang","xxx2go"]
categories: ["golang","xxx2go"]
author: "铁血执着的青春"

---
>这年头效率为王,有趁手的效率工具,将极大提升开发效率。

## 背景
golang开发过程中,我们经常需要将一些模式化的数据,转换为golang语言结构体,便于在代码中直接使用。利用一些工具,可以大大提升效率,避免认为的编辑。
目前主要的场景主要有四类:
1. curl语句转换为golang代码。
2. json语句转为golang结构体。
3. sql语句转换为golang结构体。
4. toml转换为golang结构体。

## 使用
### curl-to-go工具使用
[curl-to-go](https://mholt.github.io/curl-to-go/)

* 输入如下语句:
```
curl 'http://www.baidu.com' -H "host:www.baidu.com" -H "referer:www.baidu.com"
```

* 输出结果如下:
![
http://p.qpic.cn/qqconadmin/0/3f919eb2eaa74eeabee4878f98ff4a09/0](
http://p.qpic.cn/qqconadmin/0/3f919eb2eaa74eeabee4878f98ff4a09/0)

### json-to-go工具使用
[json-to-go](https://mholt.github.io/json-to-go/)

* 输入如下语句:
```

{"a":3,
"b":4,
"c":[1,3,4,5]
}

```

注意这里的`Inline type definitions`选项,可以配置结构体的生成采用内联方式,还是独立命名成结构体。

* 输出结果如下:
![
http://p.qpic.cn/qqconadmin/0/84c97538a7964ec489329d7cc9b8ca4e/0](
http://p.qpic.cn/qqconadmin/0/84c97538a7964ec489329d7cc9b8ca4e/0)

### sql-to-go工具使用
[sql-to-go](http://stming.cn/tool/sql2go.html)

* 输入如下ddl语句
```
CREATE TABLE Persons
(
PersonID int,
LastName varchar(255),
FirstName varchar(255),
Address varchar(255),
City varchar(255)
);
```

注意一下这里面几个有用的选项:
1. package_name用于定义生成的package名。
2. gen_json 表示生成的struct中是否需要带有json的tag。

* 输出结果如下
![
http://p.qpic.cn/qqconadmin/0/4dba612c948740088006134b25a1cacd/0](
http://p.qpic.cn/qqconadmin/0/4dba612c948740088006134b25a1cacd/0)

### toml-to-go工具使用
[toml-go-go](https://xuri.me/toml-to-go/)

* 输入如下的toml定义
```
[a]
b=3
c=[1,3,4,5]
```

输出如下的结果:
![
http://p.qpic.cn/qqconadmin/0/a41aceec32a34b7ba05448ddf13bedd3/0](
http://p.qpic.cn/qqconadmin/0/a41aceec32a34b7ba05448ddf13bedd3/0)