---
layout:     post 
title:      linux find * 和 *? 的区别
subtitle:   linux find 默认正则语法规则
date:       2018-10-23             
author:     kliang                  
header-img: img/post-bg-2015.jpg    
catalog: true                      
tags:                            
    - linux
    - 正则表达式
---

## 问题描述
在使用find朝查找东西的时候，使用* 或者 *？的时候，会匹配多多个字符，举个例子:
```
!1018 $ ls
total 0
0 a--bb-d.txt  0 a-bb-cccc-d.txt

eng@rpt-re01:~/test · 04:05 AM Tue Oct 23 ·
!1019 $ find ./ -name "a-*-*-d.txt"
./a--bb-d.txt
./a-bb-cccc-d.txt
```
普通正则表达式里面 * 的含义是匹配0 or 1 个字符，为什么上述命令可以匹配到bb-ccc呢？正常的正则表达式写法应该是`.*`才对。
如果我不想匹配到`a--bb-d.txt`，pattern应该怎么写呢？
```
!1020 $ find ./ -name "a-*?-*-d.txt"
./a-bb-cccc-d.txt
```
这里 *? 过滤掉了 -- 的文件名，为什么可以work呢？

## find 命令使用的正则匹配类型
可以通过man命令查看
```
man find (默认正则表示是用的emacs regex语法)
   -regextype type
        Changes  the  regular expression syntax understood by -regex and -iregex tests which occur later on the command line.  Currently-implemented types are emacs (this
        is the default), posix-awk, posix-basic, posix-egrep and posix-extended.
```
可以看到find默认使用的正则语法是基于emac实现的，我们去查下emas的语法说明 https://www.emacswiki.org/emacs/NonGreedyRegexp 
```
  .        any character (but newline)
  *        previous character or group, repeated 0 or more time
  +        previous character or group, repeated 1 or more time
  ?        previous character or group, repeated 0 or 1 time
```
Emacs语法里面 * 表示1个或者多个，其实等同于 .* 的用法，即贪婪匹配
.* 在文档里面的描述如下
```
The forms *?, +?, and ?? provide non-greedy versions of *, +, and ? and have been available since Emacs 21.1, released in 2001.

In Emacs 21.1 and later versions Perl-like non-greedy syntax (.*?) has been implemented.

Howto get “minimal matching” on a particular \\(.*\\) sub expression, ie. howto write a non-greedy regular expressions as in Perl (.*?):

```
所以 *？的语法是最短匹配，等同于 .*?的用法
```
# PCRE lazy
$ grep -P -o 'c.*?s' <<< 'can cats eat plants?'
can cats
```

## 结论
各个版本直接的regex的实现会有不小的差异，用的时候需要注意下，https://unix.stackexchange.com/questions/441927/how-or-why-using-is-better-than 这个link上面列出了正则语法的一些分类，推荐看一下
```
grep (assuming the GNU version) supports 4 ways to match strings:

1. Fixed strings
2. Basic regular expressions (BRE)
3. Extended regular expressions (ERE)
4. Perl-compatible regular expressions (PCRE)
grep uses BRE by default.

BRE and ERE are documented in the Regular Expressions chapter of POSIX and PCRE is documented in its official website. Please note that features and syntax may vary between implementations.

It's worth saying that neither BRE nor ERE support lazyness:

The behavior of multiple adjacent duplication symbols ( '+', '*', '?', and intervals) produces undefined results.
So if you want to use that feature, you'll need to use PCRE instead
```
