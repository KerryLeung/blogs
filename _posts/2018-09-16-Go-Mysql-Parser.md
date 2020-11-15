---
layout:     post 
title:      Golang Mysql数据map到自定义结构体     
subtitle:   通用函数
date:       2018-09-16            
author:     KL                  
header-img: img/post-bg-2015.jpg    
catalog: true                      
tags:                            
    - golang
    - mysql
    - reflect 
---

## 为什么需要一个通用函数
在项目过程中，经常会碰到读取数据库数据的需求，针对不同的数据库数据集，如果直接用mysql package本身的query-scan方法，每个不同的数据集都需要几乎类似的数据处理流程，这样有两个明显的缺点
1. 数据处理逻辑和查询行为耦合在一起
2. 代码重复度高，不利于UT
我们看一个官方的mysql package的示例代码（https://golang.org/pkg/database/sql/#example_DB_QueryContext）

```
func queryName(db *sql.DB, query string, args interface{}) {
	rows, err := db.Query(query, args...)
	if err != nil {
	        log.Fatal(err)
	}
	defer rows.Close()
	for rows.Next() {
	        var name string
	        if err := rows.Scan(&name); err != nil {
	                log.Fatal(err)
	        }
	        fmt.Printf("got name: %v\n", name)
	}
	if err := rows.Err(); err != nil {
	        log.Fatal(err)
	}
	return
}

func main() {
	db := sql.Open("mysql", dsn)
	if err := db.Ping(); err != nil {
		return
	}
	defer db.Close()
	res := queryName(db, "SELECT name FROM users WHERE age=?", 27)
	fmt.Println(res)
}
```
每个查询函数都需要判断query error，Scan error， Row error，这些都是重复出现的代码，当你查询的不是age而是其他的height
 or city的时候，已上查询流程都需要再来一遍。所以有没有办法通用的函数，可以把mysql的数据自动映射到自定义的结构体上呢？

### reflect实现通用函数
还是用上面的例子，让我们看下怎么用reflect实现数据和结构体的映射

```
import (
	"reflect"
	"fmt"
	"time"
)

func queryAnything(db *sql.DB, query string, rowStruct interface{}) {
	rows, err := db.Query(query)
	if err != nil {
	        log.Fatal(err)
	}
	defer rows.Close()
	// 确定Scan函数的输入类型
	s := reflect.ValueOf(rowStruct).Elem()
	onerow := make([]interface{}, s.NumField())
	// 按顺序遍历结构体的每个元素，取其指针值
    for i := 0; i < s.NumField(); i++ {
        onerow[i] = s.Field(i).Addr().Interface()
    }
	for rows.Next() {
	        if err := rows.Scan(onerow...); err != nil {
	                log.Fatal(err)
	        }
	        // 如果你要根据结果的数据类型做相应的处理，比如数据类型转换，比如把结果写到文件。。。
		    for i := 0; i < s.NumField(); i++ {
		    	switch s.Type().Field(i).Type.String():
		    	case "time.Time":
		    		date := s.Field(i).Interface().(time.Time)
		    		fmt.Println(date.Format("2006-01-02"))
		    	default:
		    		fmt.Println("do nothing")
		    }
		    fmt.Printf("parser succeed, result is: %v", s.Interface())
	}
	if err := rows.Err(); err != nil {
	        log.Fatal(err)
	}
	return
}

type Person struct {
	Name string
}

func main() {
	db := sql.Open("mysql", dsn)
	if err := db.Ping(); err != nil {
		return
	}
	defer db.Close()
	res = Ages
	res := queryAge(db, "SELECT name FROM users WHERE age=27", &Person{})
}
```
上面的 `queryAnything`函数结果一个结构体指针，用来动态构建Scan需要的输入参数类型列表，这里用到的reflect包的的一些结构体特性，比如结构体的元素个数，结构体field的名字等等。


上面的函数写的比较粗糙，真正项目中需要参见更多的异常判断与输入检查才可，比如对于`rowStruct`的判断，其必须是一个结构体指针，最好给 `queryAnything` func加上recovery保护，reflect的操作稍有不慎，就会引起panic从而终端运行中的进程。

同时，上面函数写法还有一个强假设
> 结构体变量定义的顺序必须与select field的顺序相同

否则会引起Scan函数的报错，大多数情况下会提示数据类型不匹配，改进的写法可以利用struct的tag写法
1. 在struct每项元素定义后面添加 tag指明该项对应的数据库column名字是什么
2. 用reflect解析struct每项field的tag找到名字，与`rows.Column()` 得到的head列表进行匹配，从而决定onerow元素的顺序

### 隔离业务逻辑与数据库查询逻辑
有了上面的通用函数之后，我们可以更进一步，对业务逻辑代码和数据库查询代码进行解耦，这样如果数据源发生变化，比如从mysql迁移到presto，这样业务代码是不需要任何变化的。
而且，可以对数据库查询函数进行一层接口封装，定义常用的interface行为，例如

```
type Queryer interface {
	Create(name, dsn string) error
	Load(query string, structPtr interface{}) error
	Insert(query string) (affectedRows, error)
	Close()
}
```
在业务代码里面，接收一个 `Queryer`参数，这样的好处是显而易见的
1. 业务代码里面凡是涉及到数据库查询行为的代码，都可以很容易UT，因为你可以很容易mock一个 `Queryer`
2. 业务代码根本不用关心底层数据库是哪个？怎么样查询的数据

