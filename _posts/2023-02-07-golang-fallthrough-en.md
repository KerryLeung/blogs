---
layout:     post
title:      "Golang's fallthrough keyword"
subtitle:   practice
date:       2023-02-07
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - golang
lang: en
ref: golang-fallthrough
---

## Background
Go has only a handful of control-flow keywords. The everyday ones are `break` and `continue`; `fallthrough` is another, but it shows up far less often. This note walks through how it is actually used.

## Official Doc
https://go.dev/ref/spec#FallthroughStmt
```
A "fallthrough" statement transfers control to the first statement of the next case clause in an expression "switch" statement. It may be used only as the final non-empty statement in such a clause.

FallthroughStmt = "fallthrough" .
The "fallthrough" statement is not permitted in a type switch.
```
That is the official explanation — let's look at a few examples of how to use the keyword.

## sample code
1. `fallthrough` must be the last statement; for example, the code below fails to compile:
```
switch {
case i > 1:
  fallthrough
  fmt.Println("I will print 1")
}
```
The error message is:
`./prog.go:13:3: fallthrough statement out of place`

2. `fallthrough` hands control to the next case clause. The trap to watch for is that it does **not** evaluate the next case's condition. For example:
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
Here `i` is 3. Without `fallthrough`, it would only print `print 1` and then exit. With the keyword:
1) the `i > 1` branch is true first, so it runs the print;
2) on hitting `fallthrough`, the next statement is forced to run even though `i = 3 > 4` is false;
3) after running the `i > 4` branch, it exits the switch.

So the output is:
```
print 1
print 4
done
```

## summary
`fallthrough` lets you force the next case branch to run after a matched switch case, so you must not rely on the next case's condition being true. Rewriting the last sample, you could do:
```
	i := 3
	switch {
	case i > 1:
		fmt.Println("print 1")
		fallthrough
	case i > 4:
	        if i > 4 {
		  fmt.Println("print 4")
	        }
	default:
		fmt.Println("default print")
	}
	fmt.Println("done")
```
For cleaner code, put `fallthrough` in the case right before `default`, and add the condition check inside the `default` branch.
