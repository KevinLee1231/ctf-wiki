# gogogo

## 题目简述

题目是 Go Web 应用加 MySQL 与 Go plugin 的组合。第一阶段利用 MySQL `COLLATE` 在不同编码表比较时的隐式转换，注册形似 `admin` 的宽字符用户名并绕过账号唯一性/事务处理问题，拿到管理员密码 hash。第二阶段利用 Go plugin 提供的读文件/HTTP 请求能力确认 Go 版本，再利用对应版本 HTTP 库的 CRLF 注入伪造上传请求，覆盖 `plugins/base.so` 并让后台重新加载 plugin 达成 RCE。

## 解题过程

### Step 1

>审计源码，发现可通过MySQL COLLATE在比较不同编码表数据时的隐式编码转换获取admin账号信息

注册ａdmin用户，会返回Duplicate key，但因为未使用事务，所以成功插入了auth表

正常登录ａdmin后表单里会返回密码hash，md5解密(密码admin123..)后登录admin用户

### Step 2

>源码中告知了go plugin两个导出函数签名，可读文件 or HTTP请求
>通过读文件可搜寻必要信息（go version)
>获知go version后可知该版本HTTP库存在CRLF注入，且后续编译go plugin需要保持版本相同
>通过HTTP请求接口CRLF注入一个上传请求，访问到上传接口，上传覆盖plugins/base.so，达成代码执行

使用题目相同的Go版本编译exp.go（或直接使用exp.so），修改exp.py中的Cookie，运行后成功覆盖base.so

### Step3

访问/admin/reload使服务重新加载plugin

然后访问

```plain
POST /admin/invoke
fn=Req&arg=CMD
```
即可RCE

## 方法总结

- 核心技巧：MySQL collation 隐式转换制造账号混淆，配合未使用事务导致的部分写入；Go HTTP CRLF 注入把插件上传请求打到内部接口并覆盖 `.so`。
- 识别信号：注册处出现数据库唯一键报错但业务未回滚、用户名可用全角/相似字符、后端暴露 Go plugin 读文件/发 HTTP 能力时，应联想到编码表比较和 SSRF/CRLF 到内部上传链。
- 复用要点：Go plugin 必须用题目相同 Go 版本编译；覆盖插件后还需要触发 `/admin/reload` 重新加载，再访问 plugin 导出的 invoke 接口执行命令。
