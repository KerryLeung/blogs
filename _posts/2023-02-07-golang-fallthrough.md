---
layout:     post 
title:      Golang fallthrough keyword
subtitle:   practice
date:       2023-02-07             
author:     KL                  
header-img: blogs/img/post-bg-desk.png    
catalog: true                      
tags:                            
    - golang
---

## Background
Golang语言中关于流程控制的关键不多，常用的有break continue，fallthrough 也是其中之一，但是较为少用，借此文档，探讨其常见用法

## Official Doc
https://go.dev/ref/spec#FallthroughStmt 
```
A "fallthrough" statement transfers control to the first statement of the next case clause in an expression "switch" statement. It may be used only as the final non-empty statement in such a clause.

FallthroughStmt = "fallthrough" .
The "fallthrough" statement is not permitted in a type switch.
```
官方解释如上，还是找一些例子看看怎么用这个关键字吧

## sample code
1. fallthrough 必须是最后一个statement，比如下面的code会报错
```
switch {
case i > 1:
  fallthrough
  fmt.Println("I will print 1")
}
```
报错信息如下
`./prog.go:13:3: fallthrough statement out of place`

2. fallthrough的功能是把分支控制权转交到下个assert statement，但是需要避坑的是：转交的时候不会管下个statement的逻辑判断是否成立，举例如下
```
	i := 3
	switch {
	case i > 1:
		fmt.Println("print 1")
		fallthrough
	case i > 4:
		fmt.Println("print 4")
	default:
		fmt.Println("default print")
	}
	fmt.Println("done")
```
例子中i的值为3，如果没有fallthrough关键字，那么只会打印`print 1` 然后退出。
加上关键字之后
1) i>1的逻辑分支首先为true，会执行打印操作
2) 遇到fallthrough 关键字，会强制执行下一个statement，即使i=3 > 4并不成立
3) 执行完i>4逻辑分之后，退出switch statement

所以上例中的输出是
```
print 1
print 4
done
```

## summary
fallthrough的关键字可以帮你执行完swtich case的匹配逻辑分支之后，强制执行下一个逻辑分支，所以用的时候不能依赖于case分支的判断
的
