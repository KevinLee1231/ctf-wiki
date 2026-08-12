# 不宽的宽字符

## 题目简述

程序运行在 Windows/Wine 环境中，目标是读取当前目录 `Z:\` 下的 `theflag`。它先用 `ReadFile` 接收 UTF-8 输入，再用 `MultiByteToWideChar(CP_UTF8, ...)` 转成 Windows 的 16 位 `wchar_t`，形成 `std::wstring filename`。随后程序故意追加宽字符串 `L"you_cant_get_the_flag"`，最后却执行：

```cpp
filename += L"you_cant_get_the_flag";
file.open((char *)filename.c_str());
```

这里没有发生字符集转换，而是把 UTF-16LE 缓冲区直接重解释成以 `NUL` 结尾的窄字符串。决定性障碍不是 Docker 或终端外壳，而是按字符编码、端序与终止符反推实际字节序列；这属于表示层编码问题，因此归入 `crypto`。

## 解题过程

### 关键观察

Windows 的 `wchar_t` 为 16 位，目标平台又是小端序。一个码点为 `0x6874` 的宽字符在内存中依次存放字节 `74 68`，被 `(char *)` 读取时就变成窄字符 `th`。同理，只要把目标窄字符串按两个字节一组解释为 UTF-16LE，就能反向构造输入。

追加内容看似使文件名无法等于 `theflag`，但窄字符串在第一个 `00` 处结束。`theflag` 共 7 个 ASCII 字节，在末尾补一个 `NUL` 后正好得到 8 字节：

```text
74 68 65 66 6c 61 67 00
  桴    晥    慬    g
```

这四个 UTF-16LE 码元分别为 `0x6874`、`0x6665`、`0x616c`、`0x0067`。程序将它们重解释为 `char *` 后看到的就是 `theflag\0`，后续追加的 `you_cant_get_the_flag` 因位于窄字符串终止符之后而完全不可见。

### 构造输入

可用 Python 从目标字节直接得到需要输入的 Unicode 字符串：

```python
payload = b"theflag\x00".decode("utf-16-le")
assert payload == "桴晥慬g"
print(payload)
```

向服务输入：

```text
桴晥慬g
```

`file.open` 最终打开当前目录中的 `theflag`，并输出其中内容。

如果目标文件名长度不便自然产生末尾的零字节，可以利用 Windows 路径中重复分隔符会被折叠的性质补齐长度。例如当前目录下偶数长度的 `flag` 可写成 `.//flag`，再按同样方式编码。

### 验证

官方源码中的数据流是 `UTF-8 输入 -> UTF-16 std::wstring -> 追加宽字符 -> 强制转换为 char * -> open`。上述 payload 在 UTF-16LE 内存中的前 8 字节严格为 `theflag\0`，因此无需依赖附件或远程地址即可验证截断成立。

## 方法总结

- 核心技巧：利用 16 位宽字符缓冲区被错误重解释为窄字符串，在小端内存中同时完成字节拼接与 `NUL` 截断。
- 识别信号：看到 `(char *)wstring.c_str()`、`reinterpret_cast<char *>` 或宽窄指针直接强转时，应先检查字符宽度、端序和终止符，而不是把它当成正常编码转换。
- 复用要点：先写出目标窄字节串并补 `\0`，再按目标平台的宽字符编码反解输入；文件名长度不合适时可借助等价路径表示调整字节数。
