# 数字幽灵城

## 题目简述

题目附件是一个 Android APK。应用启动时从资源中读取编码字符串，用自定义 Base58 码表解码，将结果写入名为 `0xGame` 的 `SharedPreferences`，最后把用户输入与其中键名为 `flag` 的值比较。

## 解题过程

反编译 `MainActivity` 后，可以确认数据流为“读取 `encodedFlag` → Base58 解码 → 写入 `SharedPreferences` → 与输入比较”。关键代码可整理为：

```java
String decodedFlag = new String(Base58.decode(getString(R.string.encodedFlag)));
SharedPreferences prefs = getSharedPreferences("0xGame", 0);
prefs.edit().putString("flag", decodedFlag).apply();

String storedFlag = prefs.getString("flag", null);
if (userInput.equals(storedFlag)) {
    // success
}
```

`SharedPreferences` 只是中间存储，不是加密机制。

继续查看 `Base58` 类，程序使用大整数逐位执行 `value = value * 58 + index`，最后调用 `toByteArray()` 得到原文。需要注意，该题的字母表顺序并非常见的 Bitcoin Base58 顺序，而是：

```text
ABCDEFGHJKLMNPQRSTUVWXYZ123456789abcdefghijkmnopqrstuvwxyz
```

资源中的密文为：

```text
RmC442S4tDMzc3CvzoCx8toKodL8SE8GRQSmz8M84k6g9jG1vVrf3c5TECZR
```

按照应用中的码表直接解码：

```python
alphabet = "ABCDEFGHJKLMNPQRSTUVWXYZ123456789abcdefghijkmnopqrstuvwxyz"
encoded = "RmC442S4tDMzc3CvzoCx8toKodL8SE8GRQSmz8M84k6g9jG1vVrf3c5TECZR"

value = 0
for char in encoded:
    value = value * 58 + alphabet.index(char)

raw = value.to_bytes((value.bit_length() + 7) // 8, "big")
print(raw.decode())
```

输出：

```text
0xGame{5f7dc2d9-5243-f706-cdbf-12e34dd3970c}
```

## 方法总结

该题的核心不是 `SharedPreferences`，而是识别并复现应用自带的 Base58 解码器。处理自定义编码时不能只根据算法名称套用在线工具，必须同时核对码表顺序、前导零处理和最终字节转换方式。
