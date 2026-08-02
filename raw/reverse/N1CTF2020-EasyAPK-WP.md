# N1CTF 2020 EasyAPK Writeup

## 题目简述

EasyAPK 是 Android APK，但决定性障碍位于 JNI 原生库中的 OLLVM 混淆和两段自定义校验，因此归入逆向而不是移动平台漏洞。Java 层只负责收集长度为 39 的输入并调用动态注册的 native 方法；第一段使用换表 Base64，第二段使用 AES-CBC。

## 解题过程

### 定位动态注册的 native 方法

解包 APK，查看 `JNI_OnLoad` 中的 `RegisterNatives` 表，可恢复 Java 方法名、签名和对应函数地址。对注册函数下断点并输入测试数据，比直接阅读 OLLVM 控制流平坦化后的伪代码更有效。

校验把花括号内内容分为两部分。用 Frida 或调试器 hook native 函数入口、AES 初始化函数和比较函数，可以记录中间缓冲区、key、IV 与目标密文，从而绕开大部分无关混淆。

### 逆转自定义 Base64

第一段目标串为：

```text
jZe3yJG3zJLHywu4otmZzwy/
```

程序使用的字符表是：

```text
abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+-/
```

它与标准 Base64 的字符顺序不同，且三个输出字节的位拼接顺序也有调整。按 native 代码逐项逆运算并去掉首尾哨兵字节，得到：

```text
17b87f9aae8933ef
```

### 解密 AES-CBC 校验块

第二段把第一段结果、固定前后缀和 `n1ctf{}` 组合成 AES 参数。调试时直接在密钥扩展前读取 16/32 字节缓冲区，可避免根据反编译器错误类型猜长度。用附件中的 32 字节目标密文执行 CBC 解密、移除 PKCS#7 padding，再去掉校验用的首字节，得到：

```text
03b5029f16f7e605
```

最终输入为：

```text
n1ctf{17b87f9aae8933ef03b5029f16f7e605}
```

重新安装 APK 或直接调用 native 校验函数，确认返回成功。

## 方法总结

APK 题不应默认停在 Java 反编译结果。看到 `RegisterNatives` 时要先恢复动态映射；遇到 OLLVM 时，动态记录算法边界的输入输出往往比清理全部控制流更省力。自定义 Base64 和 AES 都要从实际字节表、位序、key、IV 和 padding 逐项验证，不能因算法名称熟悉就套标准实现。
