# Salad for you!

## 题目简述

题目直接给出密文：

```text
GYPOFR-{t0bq_kag_tmhq_rGz}
```

标题中的 “Salad” 暗示 Caesar salad，即凯撒移位。密文仍保留了 flag 的括号、下划线和数字，因此只需要判断字母的移位量。

## 解题过程

已知正常前缀应为 `UMDCTF`。比较首字母 `G -> U`，可以确定解密时对字母向后循环移动 14 位，等价于向前移动 12 位。数字和标点保持不变：

```python
text = "GYPOFR-{t0bq_kag_tmhq_rGz}"
out = []

for char in text:
    if "a" <= char <= "z":
        out.append(chr((ord(char) - ord("a") - 12) % 26 + ord("a")))
    elif "A" <= char <= "Z":
        out.append(chr((ord(char) - ord("A") - 12) % 26 + ord("A")))
    else:
        out.append(char)

print("".join(out))
```

得到：

```text
UMDCTF-{h0pe_you_have_fUn}
```

该字符串的 SHA-256 与 README 中的摘要一致。

## 方法总结

已知格式是古典密码题中很有效的锚点。由 `GYPOFR` 与 `UMDCTF` 的对应关系可以直接确定统一移位量，不需要枚举后再凭语义猜测。实现时要分别处理大小写，并保留数字、下划线和括号。
