# Dos攻击

## panic异常导致dos<div id="dos1"></div>

panic是go的异常处理，不同于其他语言的`try`、`catch`处理机制，go必须使用`defer`函数调用`recover`函数捕获panic。如果没有捕获panic，程序会崩溃退出。

### 检测

- 查找所有可能存在panic的代码，检测是否使用defer函数捕获了异常。

## 内存泄露<div id="dos2"></div>

内存泄漏（Memory leak）是在计算机科学中，由于疏忽或错误造成程序未能释放已经不再使用的内存。内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于设计错误，导致在释放该段内存之前就失去了对该段内存的控制，从而造成了内存的浪费。

go中大部分内存泄露都是来自协程的使用不当。包括协程内部channel无限阻塞、http请求没设置超时等众多协程无法退出的场景。channel阻塞主要有四种场景：

1. 无缓冲channel进行写操作但是没有协程读
2. 有缓冲channel满了的时候进行写操作
3. 读channel时没有协程写
4. select语句所有case的操作阻塞

## 死锁<div id="dos3"></div>

go中死锁是指并发协程都在彼此等待的状态。go中有自动检测死锁的机制，如果一个程序所有协程相互等待，会导致程序崩溃退出，并且无法使用`recover`捕获。

**代码示例：**

```go
// 通一个goroutine,同一个channel读写
package main

func main(){
    ch := make(chan int)
    ch<- 1 // 阻塞，运行不到下面
    <-ch
}
```

```go
// 2个以上goroutinue，使用同一channel,读写的操作先于goroutine的创建
package main

func main(){
    ch := make(chan int)
    ch<- 1
    go func(){
        <- ch
    }()
}
```

### 检测

- 查找channel和goroutine相关代码片段，通读上下文，分析是否包含存在内存泄漏和死锁代码。

## redos<div id="dos4"></div>

go语言**原生SDK**不存在ReDOS问题，原因是普通的正则是采用递归回溯的匹配方式，而go语言使用了RE2，采用的是Thompson NFA构造法，可以保证在线性时间内匹配完成。

虽然原生SDK不存在DOS问题，但是如用引用了第三方包`golang-pkg-pcre`，会导致DOS攻击，并且与其他语言消耗计算资源不同，go中会直接导致panic崩溃退出。

**代码示例：**

```go
package main

import (
    "fmt"
    "github.com/glenn-brown/golang-pkg-pcre/src/pkg/pcre"
)

func main() {
    m := pcre.MustCompile("^(a+)+$", 0).MatcherString("aaaaaaaaaaaaaaaaaaaaaaaaaaa", 0)
    fmt.Printf("pcre 1: %v\n", m.Matches())
}
```

### 检测：

- 全局搜索关键字 pcre.MustCompile

## http慢链接<div id="dos5"></div>

go原生[http.ListenAndServe]( https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/ )函数，直接使用该函数不具备为Handler设置超时机制的能力。如果不特意设置超时，则http服务默认是无限等待客户端的。  

**正确使用示例：**

```
srv := &http.Server{
    ReadTimeout: 5 * time.Second,
    WriteTimeout: 10 * time.Second,
}
log.Println(srv.ListenAndServe())
```