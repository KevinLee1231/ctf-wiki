# Blind Mouse Challenge

## 题目简述

服务提供“加密 flag”和“加密自选消息”两个接口。算法对每个可打印字符执行：

$$
c_i=((m_i-31)\oplus i)\bmod95+31
$$

其中状态 `iv` 从 0 开始逐字符递增。加密 flag 时每次都会创建新的 `Blind`，自选消息则使用连接内持续增长的另一个实例。

## 解题过程

虽然异或后又取模使单个字符不一定能唯一逆运算，但算法是确定性的，且 flag 接口每次都从相同状态开始。先取得目标密文，再用前缀 oracle 逐字符试探。

每次猜测都新建连接，使自选消息加密器的 `iv` 重新从 0 开始；提交“已知前缀 + 候选字符”，若返回密文仍是目标密文的前缀，候选字符就正确：

```python
alphabet = "_abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
flag = "grey{"

while not flag.endswith("}"):
    for ch in alphabet + "}":
        guess = flag + ch
        encrypted = encrypt_message_on_fresh_connection(guess)
        if target.startswith(encrypted):
            flag = guess
            break
```

重复到右花括号，恢复：

```text
grey{pR3t7y_5urE_c4n_8rUT3_f0rC3}
```

## 方法总结

本题本质是确定性流状态和可选明文接口共同形成的前缀比较 oracle。取模带来的碰撞不妨碍通过更长前缀继续筛选；真正重要的是每次测试都让自选消息加密器回到与 flag 相同的初始状态。设计加密服务时，不能让秘密与攻击者输入共享可预测的密钥流，也不应暴露可直接比较的确定性密文。
