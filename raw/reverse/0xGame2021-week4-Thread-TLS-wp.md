# week4Thread_TLS

## 题目简述

程序把关键逻辑拆到两个 TLS 回调中：进程载入、进入 `main` 前先用自定义 Base64 表处理字符串 `0xgame2021`；创建校验线程时，线程入口执行前的 TLS 回调再用所得 key 对输入执行 RC4。工作线程比较结果并设置全局状态，主线程等待其结束后才输出成功或失败。

## 解题过程

TLS 回调的 `Reason` 取值应正确理解为：

```c
#define DLL_PROCESS_DETACH 0
#define DLL_PROCESS_ATTACH 1
#define DLL_THREAD_ATTACH  2
#define DLL_THREAD_DETACH  3
```

`DLL_PROCESS_ATTACH` 回调在 `main` 之前运行，`DLL_THREAD_ATTACH` 回调则在新线程的入口函数之前运行。仓库二进制中可提取到字符串 `0xgame2021` 和 65 字节改表，其中前 64 字节为字母表，最后的 `$` 是补位符：

```text
3x*Up|9n;=}LHuXcG8%taz"F_JMehE,:D(^A7f@!g`<q\2.+#PQmkoZ5yCl]Kw{[$
```

标准 Base64 编码为 `MHhnYW1lMjAyMQ==`。按相同索引替换到自定义表，并把 `=` 换成 `$`，得到 RC4 key：

```text
Hn(!_"ofHA3QHG$$
```

可用下面的代码独立复现 key 生成：

```python
import base64

standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
custom = '3x*Up|9n;=}LHuXcG8%taz"F_JMehE,:D(^A7f@!g`<q\\2.+#PQmkoZ5yCl]Kw{['

encoded = base64.b64encode(b"0xgame2021").decode()
key = encoded.rstrip("=").translate(str.maketrans(standard, custom))
key += "$" * encoded.count("=")
print(key)
```

`main` 通过 `CreateThread` 调用 `sub_411F50`，随后使用 `WaitForSingleObject` 等待线程结束；因此全局结果变量 `dword_41B270` 的读取不存在条件竞争。分析时应在 TLS 回调和 `sub_411F50` 都下断点，先确认输入缓冲区经过 RC4 后的内容，再提取 check 函数中的目标字节。RC4 加解密是同一异或过程，将目标密文用上面的 key 再执行一次 RC4 即可恢复正确输入。

官方 PDF 没有公开目标密文字节或最终 flag，仓库也只有二进制，所以这些比较常量仍需从 `sub_411F50` 中实际 dump；仅凭现有文本不能诚实给出最终字符串。

## 方法总结

本题的关键是把 Windows 装载时序纳入控制流：Process Attach TLS 生成 key，Thread Attach TLS 处理输入，线程函数完成比较，主线程等待并读取结果。原 WP 只说“改表 Base64 后做 RC4”，本版已补出回调时机、自定义表、补位规则和实际 RC4 key；最终仍需从二进制保留比较数组才能形成完整离线 EXP。
