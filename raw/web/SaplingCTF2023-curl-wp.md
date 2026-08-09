# curl

## 题目简述

这是一道 curl 基础教程。服务依次检查客户端标识、查询参数、表单 POST 数据和自定义请求头；完成所有步骤后，flag 不在响应正文，而在响应头 X-Flag 中。

## 解题过程

第一步要求 User-Agent 以 curl 开头，直接使用命令行 curl 即可。随后按服务提示依次发送：

~~~bash
curl -i "http://HOST/one?anime=chainsaw%20man"
curl -i -X POST -d "anime=jujutsu kaisen" "http://HOST/sad"
curl -i -H "X-Curl-Creator: daniel stenberg" "http://HOST/xci"
~~~

实际路径以后端提示为准；关键是分别掌握 -i、查询字符串、-d 与 -H。最后一次请求必须保留响应头，才能看到：

~~~text
X-Flag: maple{cuwurl_is_verry_fun}
~~~

仓库中的 flag.txt 写成 maple{cuwurl_is_so_fun}，但 challenge.yml 和部署环境变量都使用上面的 verrry 版本；正式平台接受值应以后两者为准。

## 方法总结

HTTP 调试不能只看正文。curl 的 -i 可同时显示状态行和响应头，-d 默认发送 application/x-www-form-urlencoded，-H 用于设置自定义头。本题还存在仓库静态文件与部署配置不一致，归档时应明确记录冲突及判定依据，不能静默选一个。
