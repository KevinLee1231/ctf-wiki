# iheartrsa

## 题目简述

服务把 flag 的 SHA-256 摘要加工成一个 257 位整数 $h$，生成两个 1024 位素数构成 $n$，并选择最小整数 $e$ 使 $h^e\ge n$。随后只给出

$$
m=h^e\bmod n
$$

与 $n$，要求在 20 秒内提交 $h$。由于 $h$ 的位数被严格限定，$e$ 和模回绕次数都很小。

## 解题过程

$2^{256}\le h<2^{257}$，而 $n$ 约为 2048 位，所以前七次幂必然小于 $n$，第八次幂越过 $n$，即 $e=8$。因此存在不大的整数 $k$ 满足

$$
h^8=m+kn.
$$

对两个 1024 位素数，保守枚举 $k<2^{10}$ 已覆盖位数给出的最坏界。逐个检查右侧是否为精确八次方：

```python
from gmpy2 import iroot

def recover_hash(m: int, n: int) -> int:
    for k in range(1, 1 << 10):
        h, exact = iroot(m + k * n, 8)
        if exact and (1 << 256) <= h < (1 << 257):
            assert int(h) ** 8 % n == m
            return int(h)
    raise ValueError("hash integer not found")
```

把恢复出的十进制整数直接提交，服务端与内部 `hsh` 比较成功并返回：

```text
tjctf{iloversaasmuchasilovemymom0xae701ebb}
```

## 方法总结

- 已知秘密整数的精确位数时，RSA 模回绕可以写成 $h^e=m+kn$，并通过枚举小商 $k$ 转回整数开根。
- 指数不必由服务显式给出；比较 $h$ 与 $n$ 的位数即可锁定最小越界指数。
- 根必须同时满足“精确幂、位数区间、模幂代回”三项校验，避免错误候选。
