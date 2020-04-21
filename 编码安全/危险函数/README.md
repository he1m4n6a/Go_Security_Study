# 危险函数

`go`的指针是不能直接运算的，所以`go`较`c`来说，相对比较安全的，因为缓冲区溢出的重要原因就是对指针地址运算时候未做严格校验。但是`go`中存在一个`unsafe`的模块，可以使用`Pointer`类型进行指针运算，允许读写任意内存，导致缓冲区溢出漏洞。

**unsafe模块主要有三个函数，两个类型：**

- func Alignof(x ArbitraryType) uintptr
- func Offsetof(x ArbitraryType) uintptr
- func Sizeof(x ArbitraryType) uintptr
- type ArbitraryType
- type Pointer

**Pointer可以对以下四种类型进行转换：**

- 任意类型的指针可以转换为Pointer
- Pointer可以转换成任意类型的指针
- uintptr可以转换成Pointer
- Pointer可以转换成uintptr

**示例代码：**

## 缓冲区溢出<div id="dangerFunc1"></div>

```go
package main

import (
   "fmt"
   "unsafe"
   "os"
   "bufio"
)

func memcpy(dst uintptr, src uintptr, len int){
    for i := 0; i < len; i++ {
        *(*int8)(unsafe.Pointer(dst)) = *(*int8)(unsafe.Pointer(src))
        dst += 1
        src += 1
    }
}

func main(){
    buf := make([]byte,16)
    fmt.Printf("Enter src string>> ");
    stdin := bufio.NewScanner(os.Stdin)
    stdin.Scan()
    src := stdin.Text()
    memcpy(*(*uintptr)(unsafe.Pointer(&buf)), *(*uintptr)(unsafe.Pointer(&src)), len(src))

    fmt.Printf("%s\n", string(buf))
}
```

## double free<div id="dangerFun2"></div>

```go
package main

/*
#include <stdlib.h>
*/
import "C"
import "unsafe"

func main() {
    cs := C.CString("test")
    C.free(unsafe.Pointer(cs))
    C.free(unsafe.Pointer(cs))
}
```

### 检测：

- 全局搜索关键字`unsafe\.Pointer`，检测是否进行了不安全的拷贝算法和多次释放。

