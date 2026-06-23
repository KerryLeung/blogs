---
layout:     post 
title:      Go fallthrough关键字 and label break
subtitle:   Go语法
date:       2019-11-26            
author:     KL                  
header-img: img/post-bg-2015.jpg    
catalog: true                      
tags:                            
    - golang
---

## fallthrough and loop
在看Go的源码的时候，经常发些loop 关键字，最近工作中也发现有代码使用fallthrough关键字，正好之前对于这两个关键字了解不多，趁机熟悉下相关用法。

先看看官方的解释
### fallthrough
`To fall through to a subsequent case, use the fallthrough keyword` https://github.com/golang/go/wiki/Switch#fall-through
```
v := 42
switch v {
case 100:
	fmt.Println(100)
	fallthrough
case 42:
	fmt.Println(42)
	fallthrough
case 1:
	fmt.Println(1)
	fallthrough
default:
	fmt.Println("default")
}
// Output:
// 42
// 1
// default
```
官方文档的例子看起来有点迷惑，v明明等于42，为什么 `case 1:` 的情况也能命中呢？ 我又自己写例子试了下，发现当 `case 42:` 命中之后，如果有fallthrough, 即继续判断case，那么无所后面case里面数字是什么都可以成功名字，这点很奇怪，没搞太明白。

#### 例子
看一个自己写的例子好了
```
	a := 4
	switch {
	case a>3:
	  fmt.Println(">3")
	  fallthrough
	case a>2:
	  fmt.Println(">2")
	default:
	  fmt.Println("default")
	}

	// output:
	// >3
	// >2
```
这个例子就比较明显了，如果 `4>3` 所以 >3 肯定会被打印，如果没有fallthrough 程序就结束了。有了这个关键字，switch会继续往下判断，`4>2` 也为真，所以又打印出了 >2, 如果在 `a>2` 里面也加上fallthrough, 那么 default也会被打印出来。

#### 注意事项
The 'fallthrough' must be the last thing in the case
官方文档如上， fallthrough 必须在case 语句中的最后一行，否则将得到异常 `./prog.go:11:4: fallthrough statement out of place`

### 总结
fallthrough 会继续switch case的判断

## label break
If there is a label, it must be that of an enclosing "for", "switch", or "select" statement, and that is the one whose execution terminates.
https://golang.org/ref/spec#Break_statements
官方文档如上，链接上面还有一个例子，但是例子不直观且没有办法直接运行。
### 例子
看一个自己写的更直观的例子好了
```

func main() {
	m, n := 2,2
	for i := 0; i < n; i++ {
		for j:=0; j< m; j++ {
			if j ==1 {
				break
			}
			fmt.Println(i,j)
		}
	}
	fmt.Println("done")
}
// output:
//   0 0
//   1 0
//   done
```
上面一个双重循环打印，非常简单，如果我想实现当 `j==1` 的时候，我直接退出整个循环怎么办呢？
众所周知，一个break只能退出一个循环，想退出多个循环，那么 label break就派上用场了, 看例子
```
func main() {
	m, n := 2,2
loop:	
	for i := 0; i < n; i++ {
		for j:=0; j< m; j++ {
			if j ==1 {
				break loop
			}
			fmt.Println(i,j)
		}
	}
	fmt.Println("done")
}
// output:
//   0 0
//   done
```
从上面例子输出来看，发现break loop 退出了整段for 循环，接下来就执行了打印done的语句，所以看到这个这里就明白了 label break的用法了。

#### 注意事项
break后面可以跟任意单词，但是要保证和循环开始前面写的一致，循环开始前的单词按照约定最好贴行首，一定要加:符号。

### 总结
label break可以简单的调出整段循环。

同样的，有label break，自然也有label continue， 用法和break类似，有兴趣的同学自己试验下喽。

