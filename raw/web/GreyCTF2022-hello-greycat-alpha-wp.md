# GreyCTF2022 - Hello GreyCat alpha

## 题目简述

应用把 `Greeting` 对象序列化到 Cookie，并以 `SHA256(secret || data)` 签名。反序列化后，数组中的键值被逐项写入环境变量，最后执行 `system('echo Hello, $name')`。前缀 MAC 可做长度扩展，环境变量又能改写 Bash 导入函数。

## 解题过程

原始 Cookie 是只含 `name=GreyCat` 的序列化对象。对 SHA-256 猜测短 secret 长度，计算合法的胶水填充与扩展摘要，在原数据后追加第二个对象；PHP 解析逻辑允许利用追加对象令 `info` 包含：

```text
BASH_FUNC_echo()=() { /readflag; }
```

长度扩展满足：已知 $H(secret\|m)$ 时，无需知道 `secret` 也能计算

$$H(secret\|m\|padding\|suffix).$$

应用验证通过后反序列化对象并调用 `putenv`。Bash 子进程启动时把 `BASH_FUNC_echo()` 导入为函数，于是原本的 `echo` 命令被攻击者函数替代，执行读 flag 程序：

```text
grey{1_c4n7_b3l13v3_3nv_v4r14bl3_15_7h15_d4n63r0u5_f7b22fd61f6f7196}
```

## 方法总结

消息认证应使用 HMAC，而不是 `hash(secret || message)`。反序列化数据中的键若能成为环境变量，相当于跨越了 PHP 到 shell 的信任边界；即使命令文本固定，`PATH`、`LD_*` 与 Bash 函数环境等变量仍可改变执行语义。
