# ZZZ

## 题目简述

程序要求输入长度为 40 的 `0xGame{...}` 字符串，并把花括号内 32 个十六进制字符解析为四个 32 位无符号整数 `x1..x4`。校验同时包含线性加法、乘法、异或和移位，最后还给出完整 flag 的 SHA-256，用于从可能的多组约束解中确定唯一结果。

## 解题过程

反编译后的关键代码如下。`strlen(s1) == '('` 中的字符常量 `'('` 数值为 40；`%8x` 则说明每个变量对应 8 个十六进制字符。

```c
int __fastcall main(int argc, const char **argv, const char **envp)
{
  unsigned int x4; // [rsp+30h] [rbp-80h] BYREF
  unsigned int x3; // [rsp+34h] [rbp-7Ch] BYREF
  unsigned int x2; // [rsp+38h] [rbp-78h] BYREF
  unsigned int x1; // [rsp+3Ch] [rbp-74h] BYREF
  char s1[112]; // [rsp+40h] [rbp-70h] BYREF

  _main(argc, argv, envp);
  puts("Input the flag:(SHA-256:4aba519d4666f5421488afaaf89efdcbe48e7a53f814ce5c1d82b46b55032651)");
  scanf("%s", s1);
  if ( !strncmp(s1, "0xGame{", 7uLL) && s1[strlen(s1) - 1] == '}' && strlen(s1) == '(' )
  {
    sscanf(s1, "0xGame{%8x%8x%8x%8x}", &x1, &x2, &x3, &x4);
    if ( 3 * x2 + 5 * x1 + 7 * x4 + 2 * x3 == -1445932505
      && 2 * (2 * (2 * x2 + x3) + x1) + x4 == -672666814
      && 7 * x2 + 3 * x1 + 5 * x4 + 4 * x3 == 958464147
      && ((x1 ^ x2) << 6) + ((x3 >> 6) ^ 0x4514) == 123074281 )
    {
      puts("Correct!");
    }
    else
    {
      puts("Wrong!");
    }
  }
  else
  {
    puts("Format error!");
  }
  return 0;
}
```

由于 C 变量类型是 `unsigned int`，运算具有 32 位溢出语义。使用 Z3 的 `BitVec(..., 32)` 可以同时保留加法溢出、异或和移位行为；若改用无限精度整数，方程语义会不同。

```python
from z3 import *
import hashlib


def sha256(s):
    return hashlib.sha256(s.encode()).hexdigest()


s = Solver()

x1, x2, x3, x4 = BitVecs("x1 x2 x3 x4", 32)

s.add(3 * x2 + 5 * x1 + 7 * x4 + 2 * x3 == -1445932505)
s.add(2 * (2 * (2 * x2 + x3) + x1) + x4 == -672666814)
s.add(7 * x2 + 3 * x1 + 5 * x4 + 4 * x3 == 958464147)
s.add(((x1 ^ x2) << 6) + ((x3 >> 6) ^ 0x4514) == 123074281)

if s.check() == sat:
    m = s.model()
    print(m)

solutions = []
while s.check() == sat:
    m = s.model()
    solutions.append(m)
    s.add(Or(x1 != m[x1], x2 != m[x2], x3 != m[x3], x4 != m[x4]))

print(f"找到 {len(solutions)} 个解")
for sol in solutions:
    X1, X2, X3, X4 = (
        sol[x1].as_long(),
        sol[x2].as_long(),
        sol[x3].as_long(),
        sol[x4].as_long(),
    )
    print(f"X1, X2, X3, X4 = {X1}, {X2}, {X3}, {X4}")
    flag = f"0xGame{{{X1:08x}{X2:08x}{X3:08x}{X4:08x}}}"
    if (
        sha256(flag)
        == "4aba519d4666f5421488afaaf89efdcbe48e7a53f814ce5c1d82b46b55032651"
    ):
        print(flag)
```

求解器可能返回多组满足四个算术约束的模型，因此脚本逐个加入阻塞条件枚举模型，再按 `%08x` 还原固定宽度十六进制字段。最后计算整个候选 flag 的 SHA-256，与程序输出的摘要比较，得到唯一正确输入。

## 方法总结

- 核心技巧：用 32 位位向量精确建模混合算术与位运算约束，再用哈希筛选模型。
- 识别信号：固定宽度整数、自然溢出、异或/移位和多条联立等式同时出现。
- 复用要点：Z3 建模必须匹配原程序位宽和有/无符号语义；格式化时保留前导零，哈希应对最终完整字符串计算。
