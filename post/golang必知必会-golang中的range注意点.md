---

title: "golang必知必会-golang中的range注意点"
date: 2020-05-04T19:48:52+08:00
lastmod: 2020-05-04T19:48:52+08:00
draft: false
tags: ["golang","range"]
categories: ["golang","range"]
author: "铁血执着的青春"

---
> range应该是golang语言中,使用最为频繁的一个关键字了,稍有不慎,可能就导致bug。本文主要针对golang语言range的使用细节进行讲解。

## 前言
如果你已经清楚的能够回答出如下几个问题,那么本篇文章或许不适合你.
那么具体由哪些问题呢？
1. golang的使用过程中,有哪些坑？
1. golang中range后面的表达式是每次都会求值的么?
2. golang中range的变量,地址是固定的么？
3. golang中range遍历slice的过程中,可以动态添加元素不? 是否会遍历到？
4. golang中range遍历map的过程中,可以动态的增删元素不？是否能够遍历到?

如果你能够清楚的回答出,上面的所有问题,那么恭喜你,range你已经掌握得非常透彻了。
本文主要是回答上述一些问题,并结合范例,来进行讲解.

主要讲解的话题如下：
1. range遍历过程中,修改值?
2. range遍历过程中,使用协程?
3. range遍历过程中,执行变更?
4. range的支持的类型?
5. range的内部实现?
6. 汇总回答上面的问题?

## range使用场景
### 1. range遍历过程中,修改值?
请求看下面的范例:
```
func changeVal(infos []Info) {

	dumpInfo(infos)

	for _, val := range infos {

		val.Age = 10
	}

	dumpInfo(infos)
}

func main() {

	var infos []Info
	infos = append(infos, Info{"a", 10})
	infos = append(infos, Info{"b", 20})
	infos = append(infos, Info{"c", 30})

	changeVal(infos)
}
```
上面的代码输出:
![http://p.qpic.cn/qqconadmin/0/bb6b23024da54cbbbecf7fb5acd5bec7/0](http://p.qpic.cn/qqconadmin/0/bb6b23024da54cbbbecf7fb5acd5bec7/0)

>总结:
>没错,在遍历非指针对象列表的时候,修改非指针的值,是没有用的。因为每一次遍历都是对列表中元素的拷贝.

### 2. range遍历过程中,使用协程?
可以参考如下代码:
 ```
 func routineTester(infos []Info) {

	for _, info := range infos {

		go routinePrinter(&info)
	}
}

func routinePrinter(info *Info) {

	time.Sleep(time.Second * 1)

	fmt.Printf("info=%+v\n", info)
}

func main() {

	var infos []Info
	infos = append(infos, Info{"a", 10})
	infos = append(infos, Info{"b", 20})
	infos = append(infos, Info{"c", 30})
    
	routineTester(infos)

	time.Sleep(time.Second * 3)
}
 ```
 上面的代码输出什么?
 答案是:
![http://p.qpic.cn/qqconadmin/0/31e915f7b1754915842ad746d7526fb6/0](http://p.qpic.cn/qqconadmin/0/31e915f7b1754915842ad746d7526fb6/0)

 没错,全是都是一样的,是不是大吃一惊。这基本上是我们使用range关键字,bug的根源.

 >range关键字使用的时候有如下特性,这里现给出结论,后面结合实现原理讲解:
 >**range中等号左边的变量,是提前定义好的,并不是临时创建.**

要注意的是 用 := 时，前面变量是被重用的，想当于第一次用的是 :=  后续循环用的是 =

所以上面的代码如果需要正确运行,可以做下面的调整：
```
func routineTester(infos []Info) {

	for _, info := range infos {

		tmpInfo := info

		go routinePrinter(&tmpInfo)
	}
}
```
没错,只需要使用一个临时变量就好,这样每个变量都是循环的时候,在栈上分配的,每次取得地址就不一样了.

### 3. range遍历过程中,追加元素?
这里还是采用例子的方式来进行说明,range的提前求值特性.
比如:
```
func evaluteInfos(infos []Info) {

	for _, info := range infos {

		infos = append(infos, info)
	}

	dumpInfo(infos)
}
```
这行代码输出什么,没错,代码输出的是:
![http://p.qpic.cn/qqconadmin/0/60d7b19dbfd646a194b39d139a6cf1df/0](http://p.qpic.cn/qqconadmin/0/60d7b19dbfd646a194b39d139a6cf1df/0)

这说明我们在遍历的时候,range infos运行的时候,已经提前将len(infos)的值计算出来的,这里的长度是3.所以最终只是会遍历3次,最终元素的个数是6.

>这里抛出另外一个结论:
>**range在遍历过程中,具有提前求值的特性,这一点在开发过程中,可以加以利用.**

**同时我们可以进行演变一下:**
1.在slice遍历过程中删除元素,可能出现panic的现象.
2.在slice遍历过程中增加元素,可能不会出现.
3.在map遍历过程中,新增元素,可能出现,也可能不出现。(根据hash值来定)
4.在map遍历过程中,删除元素,这个元素后面就不会出现了.

### 4. range的支持的类型?
![http://p.qpic.cn/qqconadmin/0/c0313e27b7ab42e4b7d31671b6e67f0a/0](http://p.qpic.cn/qqconadmin/0/c0313e27b7ab42e4b7d31671b6e67f0a/0)
对于数组来说,一个数组分配给一个array，内存级别是整个array的一次复制。
string和slice的分配只是这两个类型头，或者叫做类型元信息结构体的复制，底层数组是没变化的。
map和channel也一样，只是指针的复制。

### 5. range的内部实现?
range本质是for语句的一个语法糖。
![http://p.qpic.cn/qqconadmin/0/d45797d1d06a4294b5fa8c8429b0dfab/0](http://p.qpic.cn/qqconadmin/0/d45797d1d06a4294b5fa8c8429b0dfab/0)

业务代码实际使用的是:
INDEX和VALUE变量.

编译器会在循环开始前copy一次循环对象。编译器的编译后的逻辑：
array：
![http://p.qpic.cn/qqconadmin/0/6a393bfbc8e849d99454497532ac6a99/0](http://p.qpic.cn/qqconadmin/0/6a393bfbc8e849d99454497532ac6a99/0)

slice：
![http://p.qpic.cn/qqconadmin/0/c0a32aef1e1c41e0b2705f8099dc5ea8/0](http://p.qpic.cn/qqconadmin/0/c0a32aef1e1c41e0b2705f8099dc5ea8/0)

### 6. 汇总回答上面的问题?
结论:
1.range的左值(等号左边)是提前定义好的.取用地址是固定不变的.
2.range的右值,是提前evalute的,遍历过程中不会改变.

## 参考文档
[range循环内部实现](https://www.cnblogs.com/adarking/p/8629191.html)

[go range loop internals](https://garbagecollected.org/2017/02/22/go-range-loop-internals/)