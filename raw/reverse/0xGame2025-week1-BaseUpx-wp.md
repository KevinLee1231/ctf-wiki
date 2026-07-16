# BaseUpx

## 题目简述

附件是一个经过 UPX 压缩的 64 位 Windows PE。直接载入 IDA 时，入口附近主要是解压 stub，原程序的导入、字符串和控制流尚未恢复；脱壳后可以看到程序读取用户输入，对其进行 Base64 编码并与内置字符串比较。

## 解题过程

先检查 PE 的节名、入口点和 UPX 特征。常见信号包括 `UPX0`/`UPX1` 节、异常小的导入表以及入口处的自解压逻辑。标准 UPX 未被魔改时可直接测试并脱壳：

```bash
upx -t BaseUpx.exe
upx -d BaseUpx.exe -o BaseUpx-unpacked.exe
```

重新将脱壳文件载入 IDA，`main()` 的逻辑恢复为：读取输入、去除换行、检查长度为 46、调用 `base64_encode()`，再把编码结果与下面的内置常量比较：

```text
MHhHYW1le1cwd191XzRyM183aDNfRzBkXzBmX3VweCZiNHMzNjRfRDNzMWdufQ==
```

Base64 是可逆编码，不需要逆向其内部实现，直接解码并验证长度即可：

```python
import base64

target = "MHhHYW1le1cwd191XzRyM183aDNfRzBkXzBmX3VweCZiNHMzNjRfRDNzMWdufQ=="
flag = base64.b64decode(target)

assert len(flag) == 46
print(flag.decode())
```

输出为：

```text
0xGame{W0w_u_4r3_7h3_G0d_0f_upx&b4s364_D3s1gn}
```

## 方法总结

- 核心技巧：识别并使用 UPX 自带功能脱壳，再逆转脱壳后暴露的 Base64 比较。
- 识别信号：UPX 节名、自解压入口、混乱的初始控制流，以及脱壳后恢复的标准库调用。
- 复用要点：先确认是否为标准 UPX，能静态 `upx -d` 就不必手工 dump；脱壳只是恢复原逻辑，后续仍要分析真正的校验函数。
