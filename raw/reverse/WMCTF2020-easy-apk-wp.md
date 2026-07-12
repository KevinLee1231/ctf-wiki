# easy_apk

## 题目简述

APK 主要考察 Android 反调试和 ZUC 流密码校验。Java 层检测 debuggable、XposedInstaller、Magisk，并用 StringFog 做字符串异或混淆；Native 层检测 su、Xposed、Frida、IDA 调试。真正校验逻辑通过动态注册的 `check` 函数执行：先根据运行时检测结果恢复/派生 key，再用 AES 处理 key，最后进入 ZUC 算法生成密钥流并异或还原答案。

## 解题过程

这个apk主要考察两点一个是反调试，一个就是zuc的加密算法

### 反调试

反调试分为java层的和native层的反调试，为了稍微加强难度，几乎都采用了字符串加密的方法

#### JAVA层

主要分别进行了


1. 对APK属性的检测，是否debuggable
2. 检测是否有XposedInstaller的存在
3. 检测是否有Magisk的存在

这里的2和3的字符串都使用 StringFog 做异或加密。StringFog 的核心作用是把明文字符串在编译后替换为密文，运行时再通过固定解密函数还原；本题中跟进 `a.a` 函数即可看到异或密钥并恢复被隐藏的 Xposed/Magisk 特征字符串。项目链接仅作实现背景参考： [StringFog 项目](https://github.com/MegatronKing/StringFog)

StringFog 包装函数的关键形式如下，固定密钥为 `W!@#$M`：

```java
public static String a(String s) {
    return a.a(s, "W!@#$M");
}
```

java层的反调试可以直接通过新建一个同名工程，将lib导入工程之后直接pass掉。

接下来看看Native层的反调试

#### Native层

Native层主要检测了一些常见的root特征、Xposed以及Frida这两种Hook工具、IDA动态调试

在 `.init_proc` 函数中

函数主要执行了两种操作，


1. 第一部分主要检测了常见的su文件的存在
2. 第二部分通过字符串的动态解密，检测 `system/lib` 目录下是否存在 包含`xposed`的文件

在 `.init_array` 段中，只存在一个函数 `sub_10EFC`

这个函数最终通过 `pthread_create` 函数创建一个线程，用于检测 `frida` 的存在。

 `JNI_Onload` 函数主要通过调用 `sub_E260` 函数检测 `/proc/self/maps` 检测 `XposedBridge.jar` 的存在，然后调用 `RegisterNatives` 动态注册 `check` 函数

### 算法

算法部分主要就涉及动态注册后的 `check` 函数主要调用了zuc加密算法，key是aes加密算法后的值，在调用check函数后，会再次调用 `CheckMaps` 函数，检测Xposed的存在，同时恢复key的初始值，然后调用 `checkIDA` 函数，访问 `/proc/self/status` 检测IDA的存在，并将status的值赋给key[8],然后进行AES的加密，得到最终zuc的key值

也就是说，`check` 的关键链路是 `CheckMaps` / `checkIDA` 影响 key，随后 AES 加密得到 ZUC key，再用 ZUC 密钥流与校验数据异或还原明文。

ZUC 算法参考实现： [ZUC/lte security 参考代码](https://github.com/EinarGaustad/MasterThesis/blob/27e9285121002e1dcec1ca0d4325a6d144c3ee72/lib/src/common/liblte_security.cc)。正文需要保留的关键点是：ZUC 最终用于生成流密码密钥流，密文与密钥流异或得到明文；因此不用“逆 ZUC”，只要还原 key/iv 并生成同样的 keystream，再对校验数据异或回去即可。

最终拿到flag

```plain
W3lcomeT0WMCTF!_*Fu2^_AnT1_32E3$
```

## 方法总结

- 核心技巧：绕过 Java/Native 双层反调试，恢复 StringFog 混淆字符串，跟踪动态注册 JNI 函数中的 AES 派生 key 和 ZUC 流密码异或。
- 识别信号：APK 同时检测 debuggable、Magisk、Xposed、Frida、`/proc/self/maps`、`/proc/self/status`，且算法在 `RegisterNatives` 后才出现时，应先处理反调试和动态 JNI 注册。
- 复用要点：ZUC 作为流密码使用时“加密即解密”，关键不是逆算法，而是恢复正确 key/iv 和运行时被反调试逻辑污染的 key 字节。
