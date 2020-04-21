# 并发

## 条件竞争<div id="race1"></div>

条件竞争旨在描述一个系统或者进程的输出依赖于不受控制的事件出现顺序或者出现时机。影响也是多样的，轻则程序异常执行，重则程序崩溃。如果条件竞争漏洞被攻击者利用的话，很有可能会使得攻击者获得相应系统的特权。 

**示例代码：**

```go
package main

import (
    "sync"
    "fmt"
    "runtime"
)

var(
    // counter是所有goroutine都要增加其值的变量
    counter int

    // wg用来等待程序结束
    wg sync.WaitGroup
)

// main是所有Go程序的入口
func main(){
    runtime.GOMAXPROCS(1)
    // 计数加2，表示要等待两个goroutine
    wg.Add(2)

    // 创建两个goroutine
    go incCounter(1)
    go incCounter(2)

    // 等待goroutine结束
    wg.Wait()
    fmt.Println("Final Counter:", counter)
}

// incCounter增加包里counter变量的值
func incCounter(id int){
    // 在函数退出时调用Done来通知main函数工作已经完成
    defer wg.Done()

    for count := 0; count < 2; count++{
        // 捕获counter的值
        value := counter

        // 当前goroutine从线程退出，并放回到队列
        runtime.Gosched()

        // 增加本地value变量的值
        value++

        // 将该值保存回counter
        counter = value
    }
}
```

### 检测

- 动态测试： `go run -race main.go`

- 代码审计：
  - 并发，即至少存在两个并发执行流。这里的执行流包括线程，进程，任务等级别的执行流。golang中使用`go`关键字进行并发执行。
  - 共享对象，即多个并发流会访问同一对象。常见的共享对象有`共享内存`，`文件系统`，`信号`。在正常写代码时，这部分应该加锁。
  - 改变对象，即至少有一个控制流会**改变竞争对象的状态**。因为如果程序只是对对象进行读操作，那么并不会产生条件竞争