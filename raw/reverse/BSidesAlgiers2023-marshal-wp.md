# mArShaL

## 题目简述

题目提供 Linux 程序 `marshal` 和输出文件 `obj.marshal`。程序读取 JSON，将其序列化为 BSON，然后对第 $i$ 个字节加上下标 $i$，最后反转整个字节序列。程序还把 `ptrace(PTRACE_TRACEME)` 检查放在析构函数中：若在调试器下运行，检查失败后只输出一段 Base64 干扰信息并退出。

目标是绕开干扰分支，从源码或反汇编恢复真正的序列化流程，再对 `obj.marshal` 逆变换并解析 BSON。

## 解题过程

析构函数在 `main()` 返回后执行。其反调试分支为：

```cpp
if (ptrace(PTRACE_TRACEME, 0) < 0) {
    printf("result : eW91J3JlIGNsb3NlIGhlaGU=\n");
    exit(0);
}
```

该 Base64 解码后只是 `you're close hehe`，不是 flag。正常执行时 `ptrace` 成功，程序进入另一个分支，读取用户指定的 JSON 并调用 `json::to_bson(obj)`。

设 BSON 原始字节为 $x_i$，程序先计算 $y_i=(x_i+i)\bmod 256$，再把整个数组反转后写入文件。逆变换的顺序必须相反：先反转文件，再按原下标减去 $i$。

下面的脚本使用 PyMongo 提供的 `bson` 模块解析结果：

```python
from pathlib import Path
from bson import BSON

encoded = Path("obj.marshal").read_bytes()
shifted = encoded[::-1]
raw_bson = bytes((value - i) & 0xFF for i, value in enumerate(shifted))

obj = BSON(raw_bson).decode()
flag_hex = "".join(obj["flag"])
print(bytes.fromhex(flag_hex).decode())
```

恢复后的 BSON 长度为 98 字节，唯一字段名为 `flag`，其值是：

```text
7368656C6C6D617465737B3565637572455F4D34523568346C5F773154485F63504C5535706C55357D
```

将十六进制字符串还原为字节即可得到：

```text
shellmates{5ecurE_M4R5h4l_w1TH_cPLU5plU5}
```

## 方法总结

这道题的有效数据流是“JSON → BSON → 按位置加法 → 整体反转”。解码时严格逆序执行“整体反转 → 按位置减法 → BSON 解析 → 十六进制解码”即可。按位置变换依赖原始下标，所以如果先减再反转，下标将全部错位。

反调试代码位于析构函数也是一个分析陷阱：只观察 `main()` 会误以为程序没有主要逻辑；直接在调试器下运行又会进入红鲱鱼分支。处理带构造/析构属性的 ELF 时，应同时检查 `.init_array`、`.fini_array` 及相应符号，而不能只追踪入口函数。
