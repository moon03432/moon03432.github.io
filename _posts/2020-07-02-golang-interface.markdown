---
layout: post
title:  "xxx does not implement xxx (xxx method has pointer receiver)"
date:   2020-07-02 13:00:00 +0800
categories: Go interface
---
在开始使用 Go interface 的时候，会遇到 “xxx does not implement xxx (xxx method has pointer receiver)” 这样一个问题，我们先来看一下代码：

```go
package main

import "fmt"

type service interface {
	Serve()
}

type object struct {
	name string
}

func (o *object) Serve() {
	fmt.Printf("serving %v\n", o)
}

func main() {
	var s service = object{"hi"}
	fmt.Println(s)
}
```

go run main.go 发现以下错误：

```sh
➜  go run main.go
# command-line-arguments
./main.go:18:6: cannot use object literal (type object) as type service in assignment:
	object does not implement service (Serve method has pointer receiver)
```

原因其实很简单，因为 object 没有实现 Serve()，*object 才实现了 Serve()。只需要 s = &object 即可

```go
var s service = &object{"hi"}
```

#### 参考：

[stackoverflow](https://stackoverflow.com/questions/40823315/x-does-not-implement-y-method-has-a-pointer-receiver)  

[golang interface 原理](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-interface/)  

