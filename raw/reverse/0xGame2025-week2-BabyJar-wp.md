# BabyJar

## 题目简述

附件中的 `BabyJar.jar` 是标准 ZIP/JAR 包，清单指定入口类为 `com.BabyJar.demo.BabyJar`，包内只有两个业务类：

```text
com/BabyJar/demo/BabyJar.class
com/BabyJar/demo/Encrypt.class
```

入口类读取用户输入，调用 `Encrypt.encrypt`，再与硬编码的 Base64 字符串比较。无需运行未知 JAR，可以用 IDEA、JADX 或 JDK 自带的 `javap` 查看字节码。

## 解题过程

先列出文件并反汇编私有成员和方法：

```bash
jar tf BabyJar.jar
javap -classpath BabyJar.jar -c -p com.BabyJar.demo.BabyJar
javap -classpath BabyJar.jar -c -p com.BabyJar.demo.Encrypt
```

`BabyJar.main` 中的目标密文为：

```text
QsY1V5cX9jJyF2JSAgdikwfCEneTAgICUpNnd1Iyk8IXUkJ3QhcyZ8J3YpY=
```

`Encrypt.encrypt` 对每个输入字节执行以下步骤：

```java
int key = 0x14;
byte xored = (byte) (inputByte ^ key);
byte shuffled = (byte) (
    ((xored & 0xF0) >> 4) |
    ((xored & 0x0F) << 4)
);
```

全部字节处理完后，再调用 `Base64.getEncoder().encodeToString`。半字节高低位交换是自反操作，执行两次会恢复原值，所以逆序为 Base64 解码、再次交换半字节、异或 `0x14`：

```python
import base64

ciphertext = (
    "QsY1V5cX9jJyF2JSAgdikwfCEneTAgICUpNnd1Iyk8IXUkJ3QhcyZ8J3YpY="
)
decoded = base64.b64decode(ciphertext)

plaintext = bytearray()
for value in decoded:
    swapped = ((value >> 4) | (value << 4)) & 0xFF
    plaintext.append(swapped ^ 0x14)

print(plaintext.decode())
```

输出为：

```text
0xGame{73e214d2-d85c-4441-bc17-8e10c0e7b8c2}
```

## 方法总结

JAR 本质上是带清单和固定目录结构的 ZIP，先确认入口类和类文件数量，再恢复校验链即可。本题的顺序是“异或 → 半字节交换 → Base64”，解密必须逆序；半字节交换自身可逆，因此不需要单独设计逆函数。正文已经保留目标密文、密钥和变换逻辑，工具界面截图与外部反编译网站都不是复现所必需。
