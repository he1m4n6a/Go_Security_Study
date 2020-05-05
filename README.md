# Go_Security_Study

## 编译选项
- [x] [禁止使用低版本go sdk](./编译选项/#compile1)
- [x] [使用PIE编译](./编译选项/#compile2)

## 编码安全

### 危险函数
- [x] [缓冲区溢出](./编码安全/危险函数/#dangerFunc1)
- [x] [大整型转小整型](./编码安全/危险函数/#dangerFunc2)

### 整数溢出
- [x] [溢出数字边界](./编码安全/整数溢出/#intOverFlow1)
- [x] [大整型转小整型](./编码安全/整数溢出/#intOverFlow2)
- [x] [有符号和无符号互转](./编码安全/整数溢出/#intOverFlow3)

### Panic
- [x] [数组越界](./编码安全/Panic/#panic1)
- [x] [空指针](./编码安全/Panic/#panic2)
- [x] [除0导致panic](./编码安全/Panic/#panic3)
- [x] [GoSDK导致的panic](./编码安全/Panic/#panic4)

### 并发安全
- [x] [条件竞争](./编码安全/并发安全/#race1)

## [WEB安全](./编码安全/WEB安全/#web)

- [x] 命令注入
- [x] 代码注入
- [x] SQL注入
- [x] XML注入
- [x] XPATH注入
- [x] 模板注入
- [x] 反序列化漏洞
- [x] 文件上传漏洞
- [x] 文件下载漏洞
- [x] SSRF漏洞
- [x] XSS漏洞

## DOS攻击

- [x] [panic异常导致的dos](./DOS攻击/#dos1)
- [x] [内存泄露](./DOS攻击/#dos2)
- [x] [死锁](./DOS攻击/#dos3)
- [x] [redos](./DOS攻击/#dos4)
- [x] [http慢连接](./DOS攻击/#dos5)

## 测试工具

- [x] [IDAGolangHelper](./测试工具/#test1)
- [x] [gosec](./测试工具/#test2)
- [x] [go-fuzz](./测试工具/#test3)