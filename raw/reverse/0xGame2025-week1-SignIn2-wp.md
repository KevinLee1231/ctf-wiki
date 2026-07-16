# SignIn2

## 题目简述

程序对 ASCII 可打印字符区间 `!` 到 `~`（共 94 个字符）执行循环移位。算法与 ROT47 同类，但偏移量不是固定 47；题目给出密文，需要枚举 $0\ldots93$ 的 key，并用已知 flag 前缀筛选。

## 解题过程

密文为：

```text
@*Wq}u-guAs@}CoBo*yq!*y~*yuo##oA@F@DDIE@I/
```

反编译得到的核心函数为：

```c
void __cdecl decrypt(unsigned __int8 *flag, unsigned int key)
{
  int i; // [rsp+2Ch] [rbp-4h]

  for ( i = 0; i < strlen((const char *)flag); ++i )
  {
    if ( flag[i] > ' ' && flag[i] <= '~' )
      flag[i] = (flag[i] - key % 94 + 61) % 94 + 33;
  }
}
```

将表达式整理后，可写成 $p=(c-33-key)\bmod94+33$。字符空间只有 94 种偏移，直接穷举：

```python
ciphertext = r"@*Wq}u-guAs@}CoBo*yq!*y~*yuo##oA@F@DDIE@I/"

for key in range(94):
    plaintext = "".join(
        chr((ord(ch) - 33 - key) % 94 + 33)
        if 33 <= ord(ch) <= 126 else ch
        for ch in ciphertext
    )
    if plaintext.startswith("0xGame{"):
        print(key, plaintext)
```

偏移量为 78 时得到：

```text
0xGame{We1c0m3_2_xiaoxinxie_qq_1060449509}
```

## 方法总结

- 核心技巧：在 94 个可打印 ASCII 字符上穷举循环移位，并用已知前缀筛选。
- 识别信号：每字符独立处理、限定 `33..126`、统一加减同一 key 后取模。
- 复用要点：不要机械套用 ROT47 的固定偏移；先从代码确认字母表起点、长度和方向，再枚举完整密钥空间。
