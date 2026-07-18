# week2easyencrypt

## 题目简述

程序实现了换表 Base64：6 位分组方式与标准 Base64 相同，但索引使用自定义的 64 字符表。二进制中该表的每个字节又先与 `0x57` 异或，静态恢复表后，只需按相同索引把密文字符替换回标准字符表，再正常 Base64 解码。

恢复出的自定义表为：

```text
123DfgabcQeEFh4pklmnojqGHIJKLMNOPdRSTUVWXYZrstuvwxyz0ABCi56789+/
```

## 解题过程

自定义表与标准表之间不是按字符含义对应，而是按下标对应。例如自定义表第 $i$ 个字符表示的 6 位值仍然是 $i$。使用 `str.maketrans` 建立位置映射：

```python
import base64

standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
custom = "123DfgabcQeEFh4pklmnojqGHIJKLMNOPdRSTUVWXYZrstuvwxyz0ABCi56789+/"
cipher = "FbdWHqAUNBkwGCUvMj8xJqtUGBQxLBoBhb0="

translation = str.maketrans(custom, standard)
normalized = cipher.translate(translation)
print(normalized)
print(base64.b64decode(normalized).decode())
```

转换后的标准 Base64 为：

```text
MHhnYW1le2QwX3lvdV8xaWtlX2Jxc2U2NH0=
```

解码结果为：

```text
0xgame{d0_you_1ike_bqse64}
```

## 方法总结

- 核心技巧：恢复自定义 Base64 字符表，并按索引映射回标准表。
- 识别信号：程序每次处理 6 bit、输出长度接近 $4/3$ 倍且带 `=`，但直接 Base64 解码失败。
- 复用要点：字符表顺序决定数值，不能排序；若表本身被异或或置换，先完整恢复 64 个字符并检查唯一性。
