# Crazy Gifarfe

## 题目简述

附件能作为 GIF 正常播放，但文件尾还包含 ZIP/JAR 的目录结构。它是一个 GIFAR polyglot：GIF 解码器从文件头解释图像，JAR/ZIP 解析器从文件尾的中央目录定位归档，两种格式因此能共存在同一文件中。

## 解题过程

检查字符串或十六进制内容，可在 GIF 数据之后看到：

```text
META-INF/
META-INF/MANIFEST.MF
dFrLc/f0vtW.class
PK
```

本地附件中 GIF 签名位于偏移 `0`，第一个 ZIP 本地文件头 `PK\x03\x04` 位于偏移 `687852`，ZIP 结束目录 `PK\x05\x06` 位于偏移 `689097`。这不是简单改错扩展名，而是同一文件同时满足两种格式。

JAR 运行时从末尾中央目录找到类文件，因此可以直接执行：

```bash
java -jar crazy_gifarfe.gif
```

也可以把嵌入部分提取为 JAR 后反编译 `dFrLc/f0vtW.class`。还原出的 Java 逻辑把三段内容拼接起来：

```java
String flag = "grey{G1F4RF3_s4Y5_"
    + new String(new char[]{
        'R', 'u', '6', '_', '4', '_', '0', 'u', 'B', '_', 'd', 'U', '8', '_'
    })
    + "mY_n3Ck_1S_j3L1y}";
```

得到：

```text
grey{G1F4RF3_s4Y5_Ru6_4_0uB_dU8_mY_n3Ck_1S_j3L1y}
```

## 方法总结

- 核心技巧：根据文件尾的 `PK` 结构识别 GIF/JAR polyglot，并用对应格式的解析器读取嵌入载荷。
- 识别信号：媒体文件体积异常、`strings` 出现 `META-INF` 和 `.class`、文件尾存在 ZIP 中央目录。
- 复用要点：文件类型不能只看扩展名或开头魔数；ZIP 家族格式主要依赖尾部目录，所以很适合附加在允许尾随数据的载体后。
