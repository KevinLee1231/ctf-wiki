# N1CTF 2022 - txt

## 题目简述

题目名直接提示 DNS 的 TXT 记录。目标域名为 `n1c.tf`，需要查询其公开 DNS 元数据并从响应中读取 flag。

DNS 记录会随时间变化，因此本题的解答必须以比赛期间保存的响应为准，不能把当前查询结果反推为当年的答案。

## 解题过程

使用 `dig` 指定 TXT 类型：

```bash
dig n1c.tf TXT
```

也可以只显示记录值：

```bash
dig +short n1c.tf TXT
```

仓库保存的比赛期间响应为：

```text
n1c.tf.  300  IN  TXT  "n1ctf{n1c.tf_hacked_by_wupco}"
```

因此 flag 是：

```text
n1ctf{n1c.tf_hacked_by_wupco}
```

作为时效性复核，2026-08-02 再次查询时，TXT 值已经变为：

```text
a93e3c62cfff4cd8b6956f605e1b07ff
```

这说明比赛仓库中的历史记录是还原本题答案的必要证据，当前 DNS 状态不能替代它。

## 方法总结

域名线索不只对应网页，常见入口还包括 A、AAAA、TXT、MX、NS、CNAME、CAA 等 DNS 记录。本题只需查询 TXT，但复盘时应记录查询时间与原始响应；DNS 属于动态外部状态，过期后的实时结果可能与比赛期间完全不同。
