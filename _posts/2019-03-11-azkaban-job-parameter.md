---
layout:     post 
title:      在二进制binary的Azkaban job 之前传递参数
subtitle:   Azakaban job paramter 传递
date:       2019-03-11             
author:     KL                  
header-img: img/post-bg-2015.jpg    
catalog: true                      
tags:                            
    - azkaban
---

# 背景介绍与需求分析
## 需求
最近工作中需要用Azakaban进行任务调度，一共有3个job（A->B->C)，如果A满足一定条件的话，去触发B，B完成之后去触发C。
如果A的条件没有满足，就取消B，C的运行

## 现状
官方文档（https://azkaban.github.io/azkaban/docs/latest/ ） 真是够精简的，很多内容不够详尽。首先我大致读了下官方文档，我的job是通过 `type=command` ，常用的参数传递方式即是通过job之前的运行时环境变量的方式设置的。

要调度的job A，B，C都是一个二进制的binary，传递参数给binary是比较容易的，要从binary的运行结果中获取参数是比较费劲的。我需要传递的参数是可变，就没有办法简单在job文件里面通过设置运行环境变量的方式去传递参数了。


# 解决办法
从官方文档里面看到下面一句话
```
Parameter Output
Properties can be exported to be passed to its dependencies. A second environment variable JOB_OUTPUT_PROP_FILE is set by Azkaban. If a job writes a file to that path, Azkaban will read this file and then pass the output to the next jobs in the flow.
```
所以job之间是共享一个叫 `JOB_OUTPUT_PROP_FILE`的环境变量的，这个变量指向一个文件路径，而且文件路径是携带了execution id的，所以每次执行是独立的文件，不会冲突。
使用这个`JOB_OUTPUT_PROP_FILE`的时候有两个需要注意的地方
1. 这个文件接收json 格式的字符串
2. 凡是在文件里面定义好的变量，之后的job都可以直接调用

## 一个例子
A.job, job 是一个二进制二建
```
type=command
command=./job -step START -path $JOB_OUTPUT_PROP_FILE 
```
job 会判断下是不是到了执行job B的时间，如果满足条件，然后就开始执行B，否则就skip B C

job.go
```
package main

import (
	"fmt"
	"time"
	"flag"
	"os"

	"os/exec"
)

path := flag.String("path", "", "path to save azkaban job parameters")
step := flag.String("step", "", "step of XX")
skipDO := flag.Bool("skip do", false, "is skip do job")


func main() {
	scheduleTime, err := time.Parse("2006-01-02", "2019-03-01")
	if err != nil {
		os.Exit(1) // mark Azkaban job fail
	}

	switch *step {
	case "START":
		skip := true
		if time.Now.Format("2006-01-02") != scheduleTime {
			skip = false
		}
		cmd := exec.Command("sh", "-c", fmt.Sprintf("echo '{\"SKIP_B\": %v}'> %v", skip, *path))
		cmd.CombinedOutput()
	case "DO":
		if *skipDO {
			fmt.Println("skip, exit...")
		}
		fmt.Println("hello, ok")
	}
}
```

B.job
```
type=command
command=./job -step DO -path $JOB_OUTPUT_PROP_FILE -skipDO=${SKIP_B}
```

这里有一个小坑，Golang flag 用的时候必须 `-skipDO=${SKIP_B}` 这种方式，如果空格方式是不识别的。
