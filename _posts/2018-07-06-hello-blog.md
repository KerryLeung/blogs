---
layout:     post                    # 使用的布局（不需要改）
title:      Golang Interface{} 与 nil           # 标题 
subtitle:   interface{}的实现 #副标题
date:       2018-07-11              # 时间
author:     Kerry                   # 作者
header-img: blogs/img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - golang
    - interface 
---

## interface{}与nil的对比
### 在写代码的过程中，发现一个interface{} 参数与nil对比的一个奇怪的地方
先来看一段代码 https://play.golang.org/p/6_gUBSS57P_E
```
package main

import (
	"fmt"
	"reflect"
)

func test(i interface{}) {
	fmt.Printf("i=%v\n", i)
	fmt.Printf("isNil = %v\n", reflect.ValueOf(i).IsNil())
	if i == nil {
		fmt.Println("i is nil")
	} else {
		fmt.Println("i is not nil")
	}
	if i.(*string) == nil {
		fmt.Println("i string is nil")
	}
//	if i.(*int) == nil {
//		fmt.Println("i string is nil")
//	}
}

func main() {
	fmt.Println("Hello, playground")
	var k *string
	fmt.Printf("k==nil is %v\n", k==nil)
	test(k)
}
```
### 大家觉得预期的输出会是什么？
main函数里面`k==nil` 应该是空无疑，因为k只是声明，并没有赋值，声明操作会将k初始化为零值，k的类型是 `*string`指针，指针的零值只能是 `nil`

但是在test函数里面的 `i==nil`的判断呢？
这里看下代码的运行结果
```
Hello, playground
k==nil is true
i=<nil>
isNil = true
i is not nil
i string is nil
```
可以看到i is not nil, 但是 i string is nil, 为什么会这样呢？
## interface 变量解析
从《Go语言编程圣经》上可以看到如下描述
> 在Go语言中，变量总是被一个定义明确的值初始化，即使接口类型也不例外。对于一个接口的零值就是它的类型和值的部分都是nil（图7.1）。
![interface{}](https://ws1.sinaimg.cn/large/006tKfTcly1ft697myejwj30eg07a3z5.jpg)
所以说要判断interface{}参数是否为空，要满足type+value同时为空才行。

## 结语
当使用interface{}当函数参数的时候，如果对此参数进行非空判断，大多数情况下都是对参数的值进行非空判断，有两种方式可以实现
1. reflect方法
```reflect.ValueOf(i).IsNil()```
2. 如果确定参数类型，可以用类型推断
```i.(*string) == nil```
第二种方法要谨慎使用，比如用了例子中的```if i.(*int) == nil {```，因为参数`i`的类型在这个例子中，其type为`*string`，所以这个调用就会panic。
