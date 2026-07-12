# re1

## 题目简述

题目是 Android/Native 混合逆向。Java 层调用 `libEncrypt.so` 的 `checkYourFlag` 校验 flag。静态分析导出表中的同名函数不可逆，且调试时断不下，说明真实校验函数可能通过 JNI 动态注册。

## 解题过程

### 关键机制

在 `JNI_OnLoad` 中定位真实 `check` 函数后，可看到它把多个 libc 函数调用伪装成算法步骤：

- `strlen` 相关逻辑先做一段异或。
- `printf`、`scanf`、`strftime` 被用于实现魔改 XTEA 加密流程。
- `fgetws` 路径中存在赋值/取值逻辑。

这些 API 名称本身不是重点，实际要按调用前后的数据流恢复 key、轮函数和密文。

### 求解步骤

1. Java 层确认入口：

```java
System.loadLibrary("Encrypt");
checkYourFlag(input);
```

2. Native 层不要停在导出表的静态 `checkYourFlag`，而是跟 `JNI_OnLoad`，找到 `RegisterNatives` 传入的真实函数指针。
3. 动态调试真实 check，记录：

```text
input -> xor stage -> modified XTEA -> compare ciphertext
```

4. 提取 XTEA key、轮数、delta/魔改常量和密文，写解密脚本逆回 flag。

## 方法总结

- 识别点：导出表函数和实际 JNI 注册函数不一致。
- 分析方式：从 `JNI_OnLoad/RegisterNatives` 反查真实 native 方法。
- 加密核心：伪装在 libc API 调用序列里的魔改 XTEA，最终通过提取 key 和密文逆解。
