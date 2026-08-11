# Verification_code

## 题目简述

远端先要求完成一个 SHA-256 proof of work：在 62 个大小写字母和数字中寻找 4 字符前缀，使 `sha256(prefix + suffix)` 等于给定摘要，并要求在 60 秒内提交。通过后还需回答服务端源码中写明的 secret phrase。

## 解题过程

服务端会输出类似：

```text
sha256(XXXX + <suffix>) == <digest>
```

搜索空间大小为 $62^4=14,776,336$，可以直接枚举：

```python
import hashlib
import itertools
import string

alphabet = string.ascii_letters + string.digits

def solve_pow(suffix: str, target: str) -> str:
    for chars in itertools.product(alphabet, repeat=4):
        prefix = "".join(chars)
        digest = hashlib.sha256((prefix + suffix).encode()).hexdigest()
        if digest == target:
            return prefix
    raise ValueError("proof of work 无解")
```

脚本连接远端后解析当次的 `suffix` 和 `digest`，调用 `solve_pow` 并立即发送结果。进入第二阶段后，题目程序中可以找到固定回答：

```text
I like playing Hgame
```

提交该字符串后得到：

```text
hgame{It3Rt0O|S+I5_u$3fu1~Fo2_6rUtE-f0Rc3}
```

## 方法总结

- 核心技巧：自动解析 PoW 参数并遍历 4 位字符笛卡尔积；第二阶段直接审计服务端源码中的固定秘密。
- 识别信号：`XXXX + suffix` 表示只有 `XXXX` 未知，不能把固定后缀也放进枚举空间。
- 复用要点：哈希输入的拼接顺序、字符集和大小写必须与服务端一致；60 秒限制下应避免人工复制和重复建立连接。

> 原 PDF 没有保留最终 flag；公开参赛记录补足了结果，正文已完整保留求解过程。参考：[HGame 2020 Week2 题解](https://www.cnblogs.com/wh201906/p/12245305.html)。
