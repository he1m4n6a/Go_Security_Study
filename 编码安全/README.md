# 命令注入<div id="web"></div>

## 函数API

go中涉及系统命令执行的API

| Package | API                                                          | 功能                                                         |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| syacall | func Exec(argv0 string, argv []string,  envv []string) (err error) | 通过execve进行系统命令调用。                                 |
| syscall | func ForkExec(argv0 string, argv  []string, attr *ProcAttr) (pid int, err error) | fork and exec的组合                                          |
| syscall | func StartProcess(argv0 string, argv  []string, attr *ProcAttr) (pid int, handle uintptr, err error) | StartProcess 为软件包 os 包装 ForkExec                       |
| os      | func StartProcess(argv0 string, argv  []string, attr *ProcAttr) (pid int, handle uintptr, err error) | StartProcess 使用由 name ，argv 和 attr 指定的程序，参数和属性启动一个新进程。StartProcess 是一个低级别的界面。os/exec 软件包提供更高级的接口。 |
| os/exec | func Command(name string, arg ...string) *Cmd                | 命令返回 Cmd 结构来执行具有给定参数的命名程序。              |
| os/exec | func CommandContext(ctx context.Context,  name string, arg ...string) *Cmd | CommandContext 与 Command 相似，但包含一个上下文。如果上下文在命令完成之前完成，则提供的上下文用于终止进程（通过调用  os.Process.Kill ）。 |

## 检测

- 搜索`syscall.Exec(|syscall.ForkExec(|syscall.StartProcess(|exec.Command(|exec.CommandContext`关键字

# 代码注入

## 函数API

go语言是编译语言，运行时自身不存在代码注入，但在编译时曾存在任意代码执行漏洞[CVE-2018-6574](https://nvd.nist.gov/vuln/detail/CVE-2018-6574) 。

**go热门内嵌脚本：**

| **Project Name**                                     | **Stars** | **Forks** | **Description**                                              |
| ---------------------------------------------------- | --------- | --------- | ------------------------------------------------------------ |
| [anko](https://github.com/mattn/anko)                | 998       | 87        | Anko is a  scriptable interpreter written in Go.             |
| [gopher-lua](https://github.com/yuin/gopher-lua)     | 3400      | 369       | GopherLua: VM and  compiler for Lua in Go                    |
| [go-python](https://github.com/sbinet/go-python)     | 1000      | 112       | naive go bindings  to the CPython C-API                      |
| [go-php](https://github.com/deuill/go-php)           | 734       | 86        | PHP bindings for  the Go programming language (Golang)       |
| [go-duktape](https://github.com/olebedev/go-duktape) | 701       | 79        | Duktape  JavaScript engine bindings for Go                   |
| [go-lua](https://github.com/Shopify/go-lua)          | 1800      | 136       | A Lua VM in Go                                               |
| [golua](https://github.com/aarzilli/golua)           | 471       | 162       | Go bindings for  Lua C API - in progress                     |
| [gisp](https://github.com/jcla1/gisp)                | 437       | 324       | Simple LISP in Go                                            |
| [v8](https://github.com/augustoroman/v8)             | 315       | 63        | A Go API for the  V8 javascript engine.                      |
| [agora](https://github.com/mna/agora)                | 323       | 36        | a dynamically  typed, garbage collected, embeddable programming language built with Go |
| [purl](https://github.com/ian-kent/purl)             | 29        | 2         | Perl, but fluffy  like a cat!                                |

**go内嵌lua脚本样例：**

```go
package main

import "github.com/yuin/gopher-lua"

func main() {
    L := lua.NewState()
    defer L.Close()
    if err := L.Dostring(`print("hello world.")`); err != nil {
        panic(err)
    }
    if err := L.Dostring(`os.execute("calc")`); err != nil {
        panic(err)
    }
}
```

**CVE-2018-6574漏洞简介**

影响版本：

```
before 1.8.7
Go 1.9.x before 1.9.4
Go 1.10 pre-releases before Go 1.10rc2  
```

漏洞原理：

Go语言允许与C语言的互操作，即在Go语言中直接使用C代码。代码可以直接通过go build或go run来编译和执行，但实际编译过程中，go调用了名为cgo的工具，cgo会识别和读取Go源文件中的C元素，并将其提取后交给C编译器编译，最后与Go源码编译后的目标文件链接成一个可执行程序。gcc/clang这类的C编译器有`CFLAGS，LDFLAGS`等编译开关让开发者在编译时指定设置。cgo作为一个gcc的封装，自然也支持这类的编译开关选项。而gcc编译时，可以通过`-fplugin`指定额外的插件，gcc在编译时会加载这个插件。因此，除了在Go源码文件中可以嵌入了C代码之外，还可以指定通过`#cgo CFLAGS`指定gcc编译时的恶意插件。cgo在解析到CFLAGS关键字时，会将后面的编译选项传递给gcc。`-fplugin`指定额外的插件，可为任意的动态库，因此就获得了代码的执行权限。

这类漏洞本地执行不会有什么问题，因为自己不会引入恶意代码。但是使用`go get`导入恶意远程包的时候就存在问题。

代码示例：

```go
package main

/*
#cgo CFLAGS: -fplugin=./evil.so
#include 
void print(char *str){
	printf("%s\n",str);
}
*/

import "C"
import "unsafe"
import "fmt"

func main() {
    fmt.Println("hello world.")
    s := "cgo test"
    cs : = C.CSstring(s)
    C.print(cs)
    C.free(unsafe.Pointer(cs))
}
```

漏洞修复：

Go官方在最新版本中，加强了对编译和链接环节的检查过滤； 只允许指定的编译链接选项代入gcc执行，其他未经允许的都会被禁止。 

## 检测

- 使用`go version`查看go版本，如果版本在`before 1.8.7、Go 1.9.x before 1.9.4、Go 1.10 pre-releases before Go 1.10rc2`则存在问题。
- 搜索代码import的包，查看是否含有如下关键字`mattn/anko/|yuin/gopher-lua|sbinet/go-python|deuill/go-php|olebedev/go-duktape|Shopify/go-lua|/golua/lua|jcla1/gisp|augustoroman/v8|mna/agora|ian-kent/purl`，如果存在，进一步判断是否调用外部可控的代码执行命令。

# SQL注入

## 函数API

go自带的包database/sql提供了SQL操作的函数：

| Package      | API                                                          | 功能                                                         |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| database/sql | func (db *DB) Exec(query string, args  ...interface{}) (Result, error) | Exec 执行查询而不返回任何行。                                |
| database/sql | func (db *DB) ExecContext(ctx context.Context,  query string, args ...interface{}) (Result, error) | ExecContext 执行查询而不返回任何行。                         |
| database/sql | func (db *DB) Query(query string, args  ...interface{}) (*Rows, error) | 查询执行一个返回行的查询，通常是一个 SELECT。                |
| database/sql | func (db *DB) QueryContext(ctx  context.Context, query string, args ...interface{}) (*Rows, error) | QueryContext 执行一个返回行的查询，通常是一个 SELECT /       |
| database/sql | func (db *DB) QueryRow(query string, args  ...interface{}) *Row | QueryRow 执行一个查询，该查询最多只返回一行。QueryRow总是返回一个非零值。 |
| database/sql | func (db *DB) QueryRowContext(ctx  context.Context, query string, args ...interface{}) *Row | QueryRowContext 执行一个预计最多只返回一行的查询。QueryRowContext  总是返回一个非零值。 |

一般大家会使用go语言的第三方数据库框架，第三方框架很多，比如`gorm`，beggo框架自带的`beego orm
`等。

## 检测

- 如果使用`database/sql`原生sdk包，直接在源代码中以`.Exec|.ExecContext|.Query|.QueryContext|.QueryRow|QueryRowContext`等函数名称做关键字进行搜索，然后对这些函数执行的SQL进行排查。  
- 如果使用第三方的数据库包，查看对应的sql操作文档，使用关键字在源码中搜索，查看是否做了sql语句进行拼接。例如，使用beego框架的orm，搜索`.Raw`等关键字。

# XML注入

## 函数API

如果用户被允许输入结构化的XML片段，则他可以在XML的数据域中注入XML标签来改写目标XML文档的结构和内容，XML解析器会对注入的标签进行识别和解释，引起注入问题。encoding/xml提供XML编解码能力，编码函数有：

| API                                                          | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| func Marshal(v interface{}) ([]byte,  error)                 | Marshal 返回 v 的 XML 编码。  Marshal 通过封送每个元素来处理数组或片段。Marshal 通过编组指向的值处理指针，如果指针为零，则不写任何内容。 |
| func MarshalIndent(v interface{}, prefix,  indent string) ([]byte, error) | MarshalIndent 的工作方式与 Marshal 相同，但每个 XML 元素都以一个新的缩进行开始，该行以前缀开头，后跟一个或多个根据嵌套深度缩进的缩进副本。 |
| func (enc *Encoder) Encode(v interface{})  error             | 编码将 v 的 XML 编码写入流。                                 |
| func NewEncoder(w io.Writer) *Encoder                        | NewDecoder创建一个从r读取的新XML解析器。                     |
| func (enc *Encoder) EncodeElement(v  interface{}, start StartElement) error | EncodeElement 将 v 的 XML 编码写入流，使用 start 作为编码中最外层的标记。 |
| func (*Encoder) EncodeToken                                  | EncodeToken 将给定的 XML 令牌写入流中。如果 StartElement 和 EndElement 标记没有正确匹配，它将返回一个错误。 |

解码函数：

| API                                                          | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| func Unmarshal(data []byte, v  interface{}) error            | Unmarshal 解析 XML 编码的数据并将结果存储在v指向的值中，该值必须是任意的结构体，切片或字符串。丢弃不适合v的格式良好的数据。 |
| func (d *Decoder) Decode(v interface{})  error               | 解码像  Unmarshal 一样工作，除了它读取解码器流以查找开始元素。 |
| func NewDecoder(r io.Reader) *Decoder                        | NewDecoder 从 r 中创建一个新的 XML 分析器。如果 r 没有实现 io.ByteReader，NewDecoder 会自行缓冲。 |
| func (d *Decoder) DecodeElement(v  interface{}, start *StartElement) error | DecodeElement 的工作方式与 Unmarshal 类似，只不过它需要一个指向开始 XML 元素的指针来解码为 v。当客户端读取一些原始 XML 令牌本身时，它也很有用，但也希望延迟 Unmarshal 的某些元素。 |
| func (d *Decoder) Token() (Token, error)                     | 令牌返回输入流中的下一个XML令牌。在输入流结束时，令牌返回 nil，io.EOF。 |
| func (d *Decoder) RawToken() (Token,  error)                 | RawToken 与 Token 类似，但不验证开始和结束元素是否匹配，也不会将名称空间前缀转换为相应的 URL。 |

go语言的encoding/xml包实现了一个简单的XML 1.0 解析器，该解析器不会处理DTD声明实体，**不存在XXE漏洞。**  但第三方库可能引入xxe漏洞，例如[RobotsAndPencils/go-saml/](https://github.com/RobotsAndPencils/go-saml/issues/14)库。

## 检测：

- 查看代码import中是否有encoding/xml包，如果有，再进一步判断是否字符串拼接XML语句，存在拼接XML语句，则可能存在注入。 

- 查看代码import中是否存在go-saml 包，如果有，再进一步判断是否引入了外部实体，如果引入则可能存在xxe漏洞。

# xpath注入

go语言xpath实现由三方库提供，具体如下：

| **Project Name**                                  | **Stars** | **Forks** | **Description**                                              |
| ------------------------------------------------- | --------- | --------- | ------------------------------------------------------------ |
| [XPath](https://github.com/antchfx/xpath)         | 295       | 34        | XPath package for  Golang, supported HTML, XML, JSON query.  |
| [libxml2](https://github.com/lestrrat-go/libxml2) | 147       | 30        | Interface to  libxml2, with DOM interface.                   |
| [xmlpath](https://github.com/go-xmlpath/xmlpath)  | 102       | 41        | Strict subset of  the XPath specification for the Go language. |

上面三个库中，xpath保持更新，并且运用最广泛，仅对antchfx/xpath介绍，其具体功能实现分别是下面三个包：

- `github.com/antchfx/htmlquery`实现对html进行XPath查询，查询相关函数有：`htmlquery.Find、htmlquery.FindOne、xpath.Compile`
- `github.com/antchfx/xmlquery`实现对html进行XPath查询，查询相关函数有：` xmlquery.Find、xmlquery.FindOne、xpath.Compile
- `github.com/antchfx/jsonquery`实现对json进行XPath查询，查询相关函数有：`jsonquery.Find、jsonquery.FindOne`

如果上述这些XPath查询的语句拼接可控，则存在XPath注入。

## 检测：

- 查找import是否导入上述三个包，如果是antchfx/xpath则使用`htmlquery.Find|htmlquery.FindOne|xpath.Compile|xmlquery.Find|xmlquery.FindOne|jsonquery.Find|jsonquery.FindOne`关键字进行代码搜索找到对应的查询语句。
- 分析对应的查询语句是否引入了外不可控变量并做了拼接操作，如做了拼接则存在Xpath注入。

# 模板注入

go语言自带有两个模板包，text/template和html/template，text/template是基础包，html/template是在其上实现的专为html提供的模板。 第三方模板引擎常用的有https://github.com/flosch/pongo2/。

代码示例（dos攻击)：

```go
package main

import (
    "log"
    "net/http"
    "text/template"
)

type Inventory struct {
    Name  string
    Count int
    Func  func(int) int
    t     int
    Arr   []int
}

func Eval(i int) int {
    return i + 2
}

func main() {
    sweaters := Inventory{
        Name: "test",
        Count: 17,
        Func: func(i int) int {
            return i + 1
        },
        t:   2,
        Arr: []int{1, 2, 3, 4},
    }
    http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
        tmpl, err := template.New("test").Parse(`{{define "T0"}}{{end}}{{define "T1"}}{{template "T0"}}{{template "T0"}}{{template "T0"}}{{template "T0"}}{{template "T0"}}{{template "T0"}}{{template "T0"}}{{template "T0"}}{{template "T0"}}{{template "T0"}}{{end}}
{{define "T2"}}{{template "T1"}}{{template "T1"}}{{template "T1"}}{{template "T1"}}{{template "T1"}}{{template "T1"}}{{template "T1"}}{{template "T1"}}{{template "T1"}}{{template "T1"}}{{end}}
{{define "T3"}}{{template "T2"}}{{template "T2"}}{{template "T2"}}{{template "T2"}}{{template "T2"}}{{template "T2"}}{{template "T2"}}{{template "T2"}}{{template "T2"}}{{template "T2"}}{{end}}
{{define "T4"}}{{template "T3"}}{{template "T3"}}{{template "T3"}}{{template "T3"}}{{template "T3"}}{{template "T3"}}{{template "T3"}}{{template "T3"}}{{template "T3"}}{{template "T3"}}{{end}}
{{define "T5"}}{{template "T4"}}{{template "T4"}}{{template "T4"}}{{template "T4"}}{{template "T4"}}{{template "T4"}}{{template "T4"}}{{template "T4"}}{{template "T4"}}{{template "T4"}}{{end}}
{{define "T6"}}{{template "T5"}}{{template "T5"}}{{template "T5"}}{{template "T5"}}{{template "T5"}}{{template "T5"}}{{template "T5"}}{{template "T5"}}{{template "T5"}}{{template "T5"}}{{end}}
{{define "T7"}}{{template "T6"}}{{template "T6"}}{{template "T6"}}{{template "T6"}}{{template "T6"}}{{template "T6"}}{{template "T6"}}{{template "T6"}}{{template "T6"}}{{template "T6"}}{{end}}
{{define "T8"}}{{template "T7"}}{{template "T7"}}{{template "T7"}}{{template "T7"}}{{template "T7"}}{{template "T7"}}{{template "T7"}}{{template "T7"}}{{template "T7"}}{{template "T7"}}{{end}}
{{define "T9"}}{{template "T8"}}{{template "T8"}}{{template "T8"}}{{template "T8"}}{{template "T8"}}{{template "T8"}}{{template "T8"}}{{template "T8"}}{{template "T8"}}{{template "T8"}}{{end}}
{{define "T10"}}{{template "T9"}}{{template "T9"}}{{template "T9"}}{{template "T9"}}{{template "T9"}}{{template "T9"}}{{template "T9"}}{{template "T9"}}{{template "T9"}}{{template "T9"}}{{end}}
{{define "T11"}}{{template "T10"}}{{template "T10"}}{{template "T10"}}{{template "T10"}}{{template "T10"}}{{template "T10"}}{{template "T10"}}{{template "T10"}}{{template "T10"}}{{template "T10"}}{{end}}
{{define "T12"}}{{template "T11"}}{{template "T11"}}{{template "T11"}}{{template "T11"}}{{template "T11"}}{{template "T11"}}{{template "T11"}}{{template "T11"}}{{template "T11"}}{{template "T11"}}{{end}}
{{define "T13"}}{{template "T12"}}{{template "T12"}}{{template "T12"}}{{template "T12"}}{{template "T12"}}{{template "T12"}}{{template "T12"}}{{template "T12"}}{{template "T12"}}{{template "T12"}}{{end}}
{{define "T14"}}{{template "T13"}}{{template "T13"}}{{template "T13"}}{{template "T13"}}{{template "T13"}}{{template "T13"}}{{template "T13"}}{{template "T13"}}{{template "T13"}}{{template "T13"}}{{end}}
{{define "T15"}}{{template "T14"}}{{template "T14"}}{{template "T14"}}{{template "T14"}}{{template "T14"}}{{template "T14"}}{{template "T14"}}{{template "T14"}}{{template "T14"}}{{template "T14"}}{{end}}
{{define "T16"}}{{template "T15"}}{{template "T15"}}{{template "T15"}}{{template "T15"}}{{template "T15"}}{{template "T15"}}{{template "T15"}}{{template "T15"}}{{template "T15"}}{{template "T15"}}{{end}}
{{define "T17"}}{{template "T16"}}{{template "T16"}}{{template "T16"}}{{template "T16"}}{{template "T16"}}{{template "T16"}}{{template "T16"}}{{template "T16"}}{{template "T16"}}{{template "T16"}}{{end}}
{{define "T18"}}{{template "T17"}}{{template "T17"}}{{template "T17"}}{{template "T17"}}{{template "T17"}}{{template "T17"}}{{template "T17"}}{{template "T17"}}{{template "T17"}}{{template "T17"}}{{end}}
{{define "T19"}}{{template "T18"}}{{template "T18"}}{{template "T18"}}{{template "T18"}}{{template "T18"}}{{template "T18"}}{{template "T18"}}{{template "T18"}}{{template "T18"}}{{template "T18"}}{{end}}
{{define "T20"}}{{template "T19"}}{{template "T19"}}{{template "T19"}}{{template "T19"}}{{template "T19"}}{{template "T19"}}{{template "T19"}}{{template "T19"}}{{template "T19"}}{{template "T19"}}{{end}}
{{define "T21"}}{{template "T20"}}{{template "T20"}}{{template "T20"}}{{template "T20"}}{{template "T20"}}{{template "T20"}}{{template "T20"}}{{template "T20"}}{{template "T20"}}{{template "T20"}}{{end}}{{template "T20"}}
`)
        if err != nil {
            panic(err)
        }
        err = tmpl.Execute(w, sweaters)
        if err != nil {
            print("err")
        }
    })
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## 检测

-  搜索代码引入的import包中是否包含`html/template|text/template|flosch/pongo2`，如果存在则进一步查看调用的代码上下文是否存在解析外部变量。
-  搜索`{{|}}`关键字，查看是否是模板解析外部变量。

# 反序列化漏洞

 go语言暂时没有此方面的漏洞。

# 文件上传漏洞

```
net/http包的Request.FormFile方法
Request.MultipartForm.File方法
beego框架的Controller.GetFile方法，对上述两个函数的封装
beego框架Controller.SaveToFile方法
beego框架的Controller.GetFiles方法
```

由于go是编译型语言，所以不像php等可以上传webshell。但是如果不进行路径校验，或者检验不全还是存在通用的跨目录文件上传漏洞。

## 检测：

- 搜索 `.FormFile|.MultipartForm.File|.GetFile|SaveToFile|GetFiles`方法，查看对应的取文件名操作，如果未对路径做安全过滤，则存在跨目录文件上传漏洞。

# 文件下载漏洞

1） net/http包下的ServeFile函数：

2） BeegoOutput.Download 

3）其他文件相关API：

| API              | 功能           | 可能存在的隐患    |
| ---------------- | -------------- | ----------------- |
| os.Create        | 创建或覆写文件 | 任意文件写入      |
| os.Open          | 读取文件       | 任意文件读取      |
| os.OpenFile      | 打开文件       | 任意文件写入/读取 |
| ioutil.ReadFile  | 读取文件       | 任意文件读取      |
| ioutil.WriteFile | 创建或覆写文件 | 任意文件写入      |

## 检测

- 通过关键字`ServeFile|.Download|os.Create|os.Open|os.OpenFile|iotuil.ReadFile|ioutil.WriteFile`搜索相关读取和保存文件的操作。
- 通过使用其他的API，可通过查看路由对应的操作查看是否有相关保存文件的动作，有则进一步判断是否可以控制文件路径进行进行任意文件下载。亦可使用模糊关键字`.Down|.Save|.Write`搜索到对应的写文件操作。

# SSRF 

go语言的net库主要处理网络相关的请求，封装了几种不同协议的Client，包括http、smtp、rpc。而这些协议的Client对象并不互通，因此不能够进行跨协议的SSRF攻击。

http库相关的SSRF攻击函数：

```go
func Get(url string) (resp *Response, err error)：直接请求url

func Post(url, contentType string, body io.Reader) (resp *Response, err error)：发送POST请求，可自定义contentType和body

func PostForm(url string, data url.Values) (resp *Response, err error)：发送POST请求，只能发送form形式的请求体

func Head(url string) (resp *Response, err error)：对url发送head请求

func NewRequest(method string, url string, body string)：定制化请求
```

beego相关SSRF攻击api：

| API                                                          | 功能                                            | 可能存在的隐患                                               |
| :----------------------------------------------------------- | :---------------------------------------------- | :----------------------------------------------------------- |
| httplib.Get、httplib.Post、httplib.Put、httplib.Delete、httplib.Head | 定义对应Method的请求BeegoHTTPRequest，参数是url | URL可篡改的SSRF攻击                                          |
| BeegoHTTPRequest.Header                                      | 定义BeegoHTTPRequest的请求头                    | 请求头可篡改的SSRF攻击                                       |
| BeegoHTTPRequest.SetCookie                                   | 定义BeegoHTTPRequest的Cookie                    | Cookie可篡改的SSRF攻击                                       |
| BeegoHTTPRequest.String、BeegoHTTPRequest.Bytes、BeegoHTTPRequest.DoRequest | 发送对应的HTTP请求                              | 需要查看请求各个参数的设定是否可控，如果可控，则存在SSRF漏洞 |

## 检测：

- go中http请求的三方包非常多，可以通过搜索`http.|httplib.|.Get(|.Post(`等关键字定位到相关的http请求代码，查看上下文判断请求的url外部是否可控。

# XSS攻击

go通常用作开发后端，与前端交互的形式主要是作为api提供给前端，因此此类问题较少。但go也完全支持渲染html模板，如果使用了go内置的模板渲染功能，可能还是存在xss问题。 

go 内置了html、text/template和html/template三种html旋绕包，它们的区别：

- html (需要使用EscapeString来转义)
  https://golang.org/pkg/html/#pkg-overview
  【不推荐使用】
  仅仅对这五个字符进行编码 <, >, &, ' and ". 对一些代码注入和模板注入等无能为力。
- text/template（html需要使用HTMLEscapeString来转义，js需要JSEscapeString来转义）
  https://golang.org/pkg/text/template/#HTMLEscapeString
- html/template（html需要HTMLEscapeString来转义，js需要JSEscapeString来转义）
  https://golang.org/pkg/html/template/#HTMLEscapeString

## 检测

- 搜索代码中`html|text/template|html/template`，查看参数是否可控和做了过滤
- go内置html渲染的数据的场景来子用户的输入或者外部调用的参数等，所以可以通过`req.FormValue|req.Form.Get|r.Form|req.getParameter|http.ResponseWriter`等关键字来识别出这些场景。
