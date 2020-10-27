---
layout:     post 
title:      GOROOT，GOPATH，GOVENDOER           
subtitle:   区别于使用
date:       2018-08-18              
author:     kliang                  
header-img: img/post-bg-2015.jpg    
catalog: true                      
tags:                            
    - golang
    - GOPATH 
---

## GOPATH与GOROOT
在工作过程中，经常碰到GOPATH与GOROOT设置引起的环境变量错误，所以总结下它们的区别于用法。
可以通过 `go env` 查看下当前机器的环境变量有哪些
```!571 $ go env
GOARCH="amd64"
GOBIN=""
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GOOS="darwin"
```

Stack Overflow上有个讨论特别有用 https://stackoverflow.com/questions/7970390/what-should-be-the-values-of-gopath-and-goroot
下面的例子可以说明问题

```
directory for go related things: ~/programming/go
directory for go compiler/tools: ~/programming/go/go-1.4
directory for go software      : ~/programming/go/packages

GOROOT, GOPATH, PATH are set as following:

export GOROOT=/home/user/programming/go/go-1.4
export GOPATH=/home/user/programming/go/packages
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

### About GOROOT
你的go可执行文件安装到了哪里，默认在 `/usr/local/go`。
支持自定义安装golang位置，详情可参考官方文档https://golang.org/doc/install#install
有一点需要注意 `GOROOT must be set only when installing to a custom location`
> 只有当你把go安装到了自定义的位置的时候，必须设置GOROOT的位置，否则会出现一些奇怪的错误，比较fmt，runtime包找不到之类的.

> 同时，在开发项目的过程中，切记不要胡乱设置GOROOT，原因同上

### Abouht GOPATH
官方定义如下

```
The GOPATH environment variable specifies the location of your workspace. It defaults to a directory named go inside your home directory, so $HOME/go on Unix, $home/go on Plan 9, and %USERPROFILE%\go (usually C:\Users\YourName\go) on Windows.

If you would like to work in a different location, you will need to set GOPATH to the path to that directory. (Another common setup is to set GOPATH=$HOME.) Note that GOPATH must not be the same path as your Go installation.
```
简单而言：哪里去找到你的go项目，如果不指定，默认在 `$HOME/go`，有时会碰到一个常见的错误是 `GOPATH must not be the same path as your Go installation`, 开发项目的过程中可以注意下，如果这个错误出现，一般就是因为GOROOT设错了，或者不应该设。

GOPATH的设置与go项目的目录结构直接相关，所以先了解下golang一个项目的目录结构，参见go dochttps://golang.org/cmd/go/#hdr-GOPATH_environment_variable ，典型项目结构如下

```
GOPATH=/home/user/go

/home/user/go/
    src/
        foo/
            bar/               (go code in package bar)
                x.go
            quux/              (go code in package main)
                y.go
    bin/
        quux                   (installed command)
    pkg/
        linux_amd64/
            foo/
                bar.a          (installed package object)
    vendor/					   (go get package location)
        golang.org
        github.com
```
有一个更详细的how to start a go project 可以看这个文档 https://golang.org/doc/code.html#Introduction

### About GO GET and import package
#### import remote repo
https://golang.org/doc/code.html 能看到如下描述

```If you include the repository URL in the package's import path, go get will fetch, build, and install it automatically```
go get = got fetch + go build + go install

```
$ go get github.com/golang/example/hello
$ $GOPATH/bin/hello
Hello, Go examples!
```
如果要go get hello 这个github的package，go程序首先会去 `$GOPATH/src/github.com/golang/example/hello ` 找package，如果能找到，go get会skip `go fetch`步骤。

#### import local project repo
以上面golang项目结构为例：
`y.go` 属于main packge， `x.go` 属于 bar package，如果y.go要引用x.go里面的代码，需要做的就是在`y.go`代码里面
`import foo/bar`
一个目录一般只有一个package，import的时候不需要写绝对路径，go程序自动回去$GOPATH下面去找

### About GO_VENDOR
go vendor 是一个第三方golang包管理工具，可以查询项目用到的依赖包，查看，更新依赖包的版本 https://github.com/kardianos/govendor
安装 `go get -u github.com/kardianos/govendor`
常用命令如下，完整的文档参见github主页

```
# Setup your project.
cd $GOPATH
govendor init

# Add existing GOPATH files to vendor.
govendor add +external

# View your work.
govendor list

# Look at what is using a package
govendor list -v fmt

# Specify a specific version or revision to fetch
govendor fetch golang.org/x/net/context@a4bbce9fcae005b22ae5443f6af064d80a6f5a55
govendor fetch golang.org/x/net/context@v1   # Get latest v1.*.* tag or branch.
govendor fetch golang.org/x/net/context@=v1  # Get the tag or branch named "v1".

# Update a package to latest, given any prior version constraint
govendor fetch golang.org/x/net/context

# Format your repository only
govendor fmt +local

# Build everything in your repository only
govendor install +local

# Test your repository only
govendor test +local
```
