# 禁止使用低版本go sdk<div id="compile1"></div>

1. Go SDK在1.6版本才支持地址随机化(pie)
2. Go SDK1.12.8以前(不含)和1.11.13以前(不含)的版本存在http2 dos漏洞(CVE-2019-9512,CVE-2019-9514)

### 检测

- 使用gdb打开二进制文件，执行`p 'runtime.buildVersion'`查看go SDK版本
- 执行`strings [binary] |grep 'go1\.'`, 查看go SDK版本

# 使用PIE编译<div id="compile2"></div>

go从1.6起支持pie，但不是默认开启的。原因可能是：

1. 该选项不支持所有平台，所以不能默认开启
2. 纯go语言不存在溢出问题，大部分情况无需默认开启

未来有可能作为默认开启的选项。

```bash
go build -buildmode='pie' main.go 
```