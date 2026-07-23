# JNIdorino

## 题目简述

题目给出一个 Android APK，并提示作者“讨厌原生库，好像漏掉了某个函数调用”。反编译后可以看到 `MainActivity` 声明了大量名称随机的 JNI 方法，点击进化按钮时，`evolve()` 会逐个调用这些方法，却完全丢弃返回值，最后只弹出 `Nope!`。

真正的 flag 不在 Java/Kotlin 常量中，而藏在 `libjnidorino.so` 的一个额外 JNI 导出函数里。决定性线索不是逐个阅读数百个随机函数，而是比较“APK 声明的方法集合”和“原生库实际导出的函数集合”。

## 解题过程

### 提取两侧函数集合

用 Apktool 解包 APK，在
`smali_classes3/com/example/jnidorino/MainActivity.smali` 中收集形如下面的声明：

```smali
.method public final native oDiYusDFSQrOtyRRyBbyKchlS()Ljava/lang/String;
.end method
```

同时从 `evolve()` 的 `invoke-virtual` 指令中收集调用名。结果为：

```text
native 声明：500 个
evolve 中涉及的不同 native 方法：500 个
```

接着查看 ARM64 原生库的动态符号：

```bash
readelf -Ws libjnidorino.so \
  | grep 'Java_com_example_jnidorino_MainActivity_'
```

库中共有 501 个对应 `MainActivity` 的 JNI 导出。对两边名称排序并做集合差分，唯一只存在于原生库中的函数是：

```text
Java_com_example_jnidorino_MainActivity_GhLagoBPGjAdjYQldkhrYdgky
```

这就是作者“忘记调用”的函数。随机方法数量很大只是为了掩盖这个一项差异。

### 还原额外函数的返回值

定位该符号后反汇编，可以看到函数通过 `adrp` 与 `add` 取
`.rodata` 中地址 `0x25570` 的字符串，再将它转换为 Java `String` 返回。该位置的内容为：

```text
VU1EQ1RGe2wwdjNfbkB0MXZlX2ZVbnN9
```

它是标准 Base64。解码：

```bash
printf '%s' 'VU1EQ1RGe2wwdjNfbkB0MXZlX2ZVbnN9' | base64 -d
```

得到：

```text
UMDCTF{l0v3_n@t1ve_fUns}
```

也可以修改 Smali，为额外函数补上 native 声明与调用，再把返回值放进界面；但静态集合差分已经足以完成题目，无需重打包 APK。

## 方法总结

- 核心技巧：将托管层 native 声明、实际调用和 ELF 导出符号视为三个集合，优先做集合差分。
- 识别信号：大量同构、随机命名的 JNI 包装通常是噪声；题目若提示“漏掉一个函数”，应先比较数量与名称，而不是逐函数逆向。
- 复用要点：JNI 静态导出通常采用 `Java_<包名>_<类名>_<方法名>` 命名；定位异常符号后，再跟踪它引用的 `.rodata` 和返回值构造过程。
