## IDAGolangHelper<div id="test1"></div>

IDAGolangHelper是IDA的一个插件，是前面的逆向工具中的GolangAssist的升级版本。在没有源码，并且去掉了符号表的情况下，可以利用IDAGolangHelper插件在ida中辅助分析，恢复一些调试符号信息。

具体使用方式参考：https://github.com/sibears/IDAGolangHelper 

## gosec<div id="test2"></div>

gosec是一个Go语言源码安全分析工具，其通过扫描Go AST（抽象语法树）来检查源代码是否存在安全问题，目前可用的扫描规则如下：

G101：查找硬编码凭证

G102：绑定到所有接口

G103：审计不安全区块的使用

G104：审计错误未检查

G105：审计math/big.Int.Exp的使用

G106：审计ssh.InsecureIgnoreHostKey的使用

G201：SQL查询构造使用格式字符串

G202：SQL查询构造使用字符串连接

G203：在HTML模板中使用未转义的数据

G204：审计命令执行情况

G301：创建目录时文件权限分配不合理

G302：chmod文件权限分配不合理

G303：使用可预测的路径创建临时文件

G304：作为污点输入提供的文件路径

G305：提取zip存档时遍历文件

G401：检测DES，RC4或MD5的使用情况

G402：查找错误的TLS连接设置

G403：确保最小RSA密钥长度为2048位

G404：不安全的随机数源（rand）

G501：导入黑名单列表：crypto/md5

G502：导入黑名单列表：crypto/des

G503：导入黑名单列表：crypto/rc4

G504：导入黑名单列表：net/http/cgi

**具体使用方法参考：**

https://github.com/securego/gosec

https://github.com/re4lity/Hacking-With-Golang

## go-fuzz<div id="test3"></div>

go-fuzz是一款基于go语言源码的随机测试(Random testing)工具。

具体原理和使用方法参考：https://github.com/dvyukov/go-fuzz。