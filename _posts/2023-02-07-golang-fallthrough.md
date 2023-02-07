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
```
官方解释如上
