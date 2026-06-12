# appfriend

## 题目简述

题目是 Android native 逆向。Java 层入口最终进入 native so，核心算法是 SM4；so 的 init 段还有检测/反调试逻辑，需要先绕过检测，再从 native 代码中提取 SM4 key 和密文进行解密。

附件只给出 `appfriend.apk`，因此分析入口是 APK 解包后的 Java 层调用链和 `lib/*.so` native 库。解题时不能只看 Java 伪代码，真正的校验数据在 native 侧，截图中的 key、密文和解密结果都需要转写到正文，避免关键信息只保存在图片中。

## 解题过程

1. Java层入口定位

![1. Java层入口定位](<WMCTF2025-appfriend-wp/1-java层入口定位.png>)

发现是native 逆向，解包分析so

![发现是native 逆向，解包分析so](<WMCTF2025-appfriend-wp/发现是native-逆向-解包分析so.png>)

看到是sm4

绕过检测（在init段）

![绕过检测（在init段](<WMCTF2025-appfriend-wp/绕过检测-在init段.png>)

提取秘钥

```
01 23 45 67 89 ab cd ef,fe dc ba 98 76 54 32 10
```

提取密文

```
db e9 8e 0a d4 7e d6 58 74 1c d3 8e f8 59 59 85 81 77 d9 f3 a8 f9 0f 24 cf e1 4f d1 1a 31 3b 72 00 2a 8a 4e fa 86 3c ca d0 24 ac 03 00 bb 40 d2
```

进行解密

![进行解密](<WMCTF2025-appfriend-wp/进行解密.png>)

解题主线是：从 Java 入口定位 native 调用，解包 so 后识别 SM4 常量和调用链；绕过 init 段检测后，直接提取 16 字节 key 和密文字节。这里的 key 由两段 8 字节组成：

```text
01 23 45 67 89 ab cd ef fe dc ba 98 76 54 32 10
```

密文为 48 字节，按 SM4 分组解密即可得到结果。

最终明文为：

```text
WMCTF{sm4_1s_1imsss_test_easyss}
```

## 方法总结

- 核心技巧：Android Java 层定位入口，native so 中识别 SM4，绕过 init 段检测后静态提取 key/cipher。
- 识别信号：Java 层逻辑很薄、关键校验进入 native，且 so 中出现 SM4 常量或轮函数结构时，应转向 native 加密算法恢复。
- 复用要点：不要只保留截图，WP 中必须写明 key、密文和算法；init 段检测会影响调试或执行流程，提取前先确认检测是否需要 patch。
