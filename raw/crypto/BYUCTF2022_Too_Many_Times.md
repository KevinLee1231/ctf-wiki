# BYUCTF 2022 - Too Many Times

## 题目简述

题目给出三段等长、仅含大写字母的密文，并暗示一次一密被使用了“太多次”。三个明文共用了同一个 31 字母密钥，因此完美保密条件已经被破坏。

## 解题过程

字母型 OTP 可视为模 26 加法。若把三段 31 字符密文首尾拼接，共用的密钥就会每 31 字符重复一次，整个问题等价于密钥长度为 31 的 Vigenere：

```text
LKAHEIIWPQPGQCUOXHIQFIUBXEHJBRU
RQXJTVSYVJVDJPASBLDQGIWWHPWISKT
DTKXPPECBFVHWLIEQXDNCFWCIYPXYFB
```

对拼接结果做频率分析可得到近似密钥 `KMGFLEELPFEDQLABXPPWOEWJEWPEQRB`，相应明文已经出现 `BYUCTE...`、`HEREI...` 等明显英文结构。随后利用已知格式 `BYUCTF` 和上下文逐列修正密钥：同一列的密钥改动会同时影响三条明文，可据另外两条是否仍像英文来排除错误。

最终密钥为：

```text
KMGFLDAKJFPDSOAMXSPWOERIQWPEQRO
```

三段明文恢复为连续英文，其中第一段开头为 `BYUCTFIMGLADYOUCAPTUREDTHISFLAG`。按赛事格式还原得到：

```text
byuctf{imgladyoucapturedthisflag}
```

## 方法总结

重复使用 OTP 会把问题降级为重复密钥密码。将等长密文拼接成 Vigenere 是方便的工程化视角；真正有效的校正依据是 flag 已知前缀、自然语言统计以及多条明文之间的同列约束。
