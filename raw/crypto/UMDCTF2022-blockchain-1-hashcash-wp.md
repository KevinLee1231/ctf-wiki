# UMDCTF2022 Blockchain 1 - Hashcash Writeup

## 题目简述

服务要求提交一个符合 Hashcash v1 外形的邮件头，并使其 SHA-1 摘要具有 20 个前导零位。虽然题名带有 Blockchain，但服务端没有区块、交易或链上状态；真正障碍是构造哈希工作量证明，因此归入 `crypto`。

服务端接受七段格式：

```text
1:20:YYMMDD:<白名单邮箱>::<Base64 nonce>:<Base64 counter>
```

日期必须在服务器当前时间前后两天内，邮箱用户名必须位于白名单。

## 解题过程

SHA-1 摘要以十六进制表示时，一个十六进制零等于四个零位。因此 20 个前导零位等价于：

```python
hashlib.sha1(header.encode()).hexdigest().startswith("00000")
```

选择稳定存在的 `gary@hashcash.com`，生成随机 nonce，然后枚举计数器。nonce 和 counter 字段只要求能通过 Base64 解码，不需要具有固定长度：

```python
import base64
import datetime
import hashlib
import os

date = datetime.date.today().strftime("%y%m%d")
nonce = base64.b64encode(os.urandom(5)).decode()

for counter in range(1 << 30):
    encoded_counter = base64.b64encode(str(counter).encode()).decode()
    header = (
        f"1:20:{date}:gary@hashcash.com::"
        f"{nonce}:{encoded_counter}"
    )
    if hashlib.sha1(header.encode()).hexdigest().startswith("00000"):
        print(header)
        break
```

连接服务后先发送 `y`，再把找到的完整 header 发送给 `X-Hashcash:` 提示。服务端依次校验字段数、日期、邮箱和 SHA-1 前缀，成功后返回：

```text
UMDCTF{H@sh_c4sH_1s_th3_F@th3r_0f_pr00f_0f_w0rk}
```

## 方法总结

分类应以决定性机制为准，而不是题名。Hashcash 是比特币工作量证明的历史先驱，但本题没有区块链语义。实现时还要区分“前导零位”和“前导零十六进制字符”：题目要求 20 位，对应恰好 5 个十六进制零，平均需要约 $2^{20}$ 次尝试。
