# decrypt then eval

## 题目简述

服务为每次输入重新以同一把 `KEY`、同一 `IV` 创建 AES-CFB-128 对象，然后对玩家提供的密文解密并执行 Python `eval`：

```python
aes = AES.new(KEY, AES.MODE_CFB, IV, segment_size=128)
print(eval(aes.decrypt(ct)))
```

异常被折叠为 `invalid ct!`。CFB 在首块满足 $P=C\oplus E_K(IV)$，而服务重复使用 IV 使该首块掩码在查询间不变；再加上 `eval` 的成功/失败反馈，攻击者可先取得一段已知明文对应的密文，再把它改写成任意同长度 Python 表达式。根本问题是未认证加密的可篡改性与危险的 `eval` 叠加。

## 解题过程

先把异常输出作为有效性 oracle：逐个尝试一字节密文，寻找会被解成合法数字字面量的连续候选，从而定位某个已知单字节明文对应的密文。之后逐字节延长候选，并对末字节施加已知 XOR 差分；官方 solver 用数值字面量和复数字面量后缀 `j` 进行双重有效性测试，最终构造出一个长度为 11 的可 `eval` 已知明文 $P$。

一旦有这一对 $(C,P)$，首 CFB 块的 keystream 为 $S=C\oplus P$。目标载荷取 `print(FLAG)`，于是应发送：

$$
C_{\mathrm{target}}=C\oplus P\oplus\texttt{b"print(FLAG)"}.
$$

因为两个明文长度均为 11 字节，服务解密结果恰为 `print(FLAG)`，`eval` 执行它并由外层 `print` 输出返回值。官方 `solve/solv.py` 的最终注入正是：

```python
conn.sendlineafter(
    b"ct: ",
    xor(z, b"print(FLAG)", str(n).encode() + b"0" * 10).hex().encode(),
)
```

其中 `z` 是上述 oracle 构造的密文，第三个参数是对应已知字面量；脚本枚举首位数字以匹配实际构造出的 $P$。执行成功后得到：

```text
DUCTF{should_have_used_authenticated_encryption!}
```

## 方法总结

保密模式不提供完整性：已知明文的 CFB 密文可按 XOR 差分改成任意等长明文。认证加密（例如 AES-GCM、ChaCha20-Poly1305）会在执行前拒绝篡改密文；但即便使用了认证，也绝不能对攻击者可控数据执行 `eval`。错误信息本身也应避免成为可重复利用的语法有效性 oracle。
