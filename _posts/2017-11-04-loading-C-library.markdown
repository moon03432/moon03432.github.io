---
layout: post
title:  "loading C library"
date:   2017-11-04 13:00:00 +0800
categories: coding C Go Python
---
不同语言有不同的特性，C/C++ 最底层，Python 的语法糖编程效率高，Golang 的并发编程和模型写起来很舒服。在很多实际项目中也有 Python 以及 Golang 调用 C 动态库的例子。本文记录了 C, Go, Python 语言是如何加载 C 动态库的。

首先，先写一个 C 乘法库：

```c
// cmult.h
float cmult(int int_param, float float_param);
```

```c
// cmult.c
#include <stdio.h>
#include "cmult.h"

float cmult(int int_param, float float_param) {
    float return_value = int_param * float_param;
    printf("    In cmult : int: %d float %.1f returning %.1f\n", int_param, float_param, return_value);
    return return_value;
}
```

编译：

```sh
gcc -fPIC -shared -o libcmult.so cmult.c
```

先来看 C 语言的加载：

```c
// main.c
#include <stdio.h>
#include "cmult.h"

int main(void) {
    float res = cmult(2, 1.1);
	  printf("result is %f\n", res);
    return 0;
}
```

编译：

```sh
gcc main.c -I<path_to_cmult.h> -lcmult -L<path_to_libcmult.so> -o main
```

测试:

```sh
➜ ./main
This is a shared library test...
    In cmult : int: 2 float 1.1 returning 2.2
result is 2.200000
```

然后是 Python，使用了Python标准库中的ctypes：

```py
# ctypes_test.py
import ctypes
import pathlib

if __name__ == "__main__":
    libname = "/usr/local/lib/libcmult.so"
    c_lib = ctypes.CDLL(libname)

    x,y = 6, 2.3
    c_lib.cmult.restype = ctypes.c_float
    answer = c_lib.cmult(x,ctypes.c_float(y))
    print(f"    In Python: int: {x} float {y:.1f} returning {answer:.1f}")
```

测试：

```sh
➜  python3 ctypes_test.py
    In cmult : int: 6 float 2.3 returning 13.8
    In Python: int: 6 float 2.3 returning 13.8
```

最后是 Go，使用了 cgo:

```go
package main

// #cgo LDFLAGS: -lcmult
// #include <cmult.h>
import "C"
import "fmt"

func main() {
	res := C.cmult(2,1.1)
	fmt.Println(res)
}
```

测试：

```sh
➜  go run main.go
    In cmult : int: 2 float 1.1 returning 2.2
2.2
```



#### 相关资料：

[cgo](https://blog.golang.org/cgo)  

[cgo is not go](https://dave.cheney.net/2016/01/18/cgo-is-not-go)  

[ctypes](https://docs.python.org/3/library/ctypes.html)

