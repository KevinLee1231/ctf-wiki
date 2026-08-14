# bi0sCTF 2024 - ezv8

## 题目简述

服务接收一段不超过 1 MiB 的 JavaScript，写入临时文件后直接执行 `./d8 <file>`。题目原本附带一个 V8 编译器补丁，预期通过 JIT 类型混淆取得内存破坏；比赛实例却保留了 d8 的 `load()` 宿主函数，且进程工作目录中存在可读的 `flag.txt`，形成更短的非预期解。

## 解题过程

服务端包装器没有为 d8 启用限制文件访问的选项，也没有删除 shell 内建函数。d8 的 `load(path)` 会读取并执行指定 JavaScript 文件，因此提交以下内容即可让进程打开 flag 文件：

```javascript
load("flag.txt");
```

连接脚本只需按协议先发送源码长度，再发送源码本身：

```python
payload = b'load("flag.txt");\n'
io.sendlineafter(b"File size >> ", str(len(payload)).encode())
io.sendafter(b"Data >> ", payload)
io.interactive()
```

`flag.txt` 的内容并不是合法、已定义的 JavaScript 程序。d8 在解析或执行它时会输出包含文件名、源码位置和未定义标识符的错误信息，错误文本因此泄漏完整 flag。即使目标文件改成 `print("...")` 形式，`load()` 也会直接执行并打印，根因仍是未限制的本地文件加载。

仓库中的 `v8.patch` 注释掉 `InferMapsUnsafe` 对 `JSCreate` 副作用的“不可靠 map”标记，这确实能产生预期的 JIT 利用链，但对原版 `ezv8` 没有必要。该利用在后续 `ezv8 revenge` 中才是必须步骤；本篇只记录比赛服务上最短且由官方 README 明确认可的解法。

## 方法总结

运行自定义 JavaScript 引擎题时，应先枚举宿主暴露的 shell 内建函数，再分析复杂的引擎漏洞。这里 `load()` 直接跨越了预期的内存安全边界，能够读取工作目录中的 flag；错误回显又把文件内容带回客户端。修复方向是构建不含文件加载能力的 shell，或在独立目录/沙箱中运行且不给 flag 文件读取权限。
