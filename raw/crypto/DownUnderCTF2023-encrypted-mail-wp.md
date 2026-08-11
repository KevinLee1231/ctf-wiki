# DownUnderCTF 2023 encrypted mail Writeup

## 题目简述

邮件系统组合了离散对数认证、ElGamal 密钥封装、自定义 128 位流密码和 ElGamal 签名。`admin` 会给新用户发送欢迎邮件；`flag_haver` 只在收到由 `admin` 签名且解密为 `Send flag to <用户名>` 的消息时发送 flag。漏洞链由可预测认证随机数、线性流密码和未绑定收件人的签名组成。

## 解题过程

登录挑战每轮生成 $(c_1,c_2)$。其中 $c_1=g^r$；当随机位 $b=0$ 时 $c_2=g^c$，当 $b=1$ 时 $c_2=y^r$。

给自己注册已知私钥 $x$ 的账号后，可用 $c_1^x\stackrel{?}=c_2$ 判断每一位 $b$。这些位来自 Python 全局 `random` 的 `round(random.random())`。每轮随后两次 `getrandbits(768)`，使相邻观测之间固定跨过 50 个 MT19937 输出。预计算每个状态位对观测最高位的线性影响，再收集 $156\times128=19968$ 个观测，就能在 $\operatorname{GF}(2)$ 上解出完整 MT 状态。

```python
# images[i] 是第 i 个 MT 状态基向量对全部观测位的影响位置。
rows = []
for state_bit in range(624 * 32):
    row = [0] * (624 * 32)
    for output_bit in images[state_bit]:
        row[output_bit] = 1
    rows.append([row[0]] + row[32:])

matrix = Matrix(GF(2), rows).augment(vector(observed_bits))
solution = matrix.right_kernel_matrix()[0][:-1]
```

恢复状态后，可以预测任意账号的 128 个认证答案，因此能以 `admin` 身份登录，但这仍没有管理员的签名私钥。下一步复用一封真实欢迎邮件：签名只覆盖流密码密文 `ct`，没有绑定 ElGamal 封装的对称密钥，也没有绑定收件人。

自定义流密码的状态更新和输出位都只由移位、异或与选定位相加后取最低位组成，在 $\operatorname{GF}(2)$ 上是线性的。对一封长度合适的管理员欢迎密文建立符号密钥流方程，要求解密前缀等于：

```text
Send flag to
```

并约束余下 8 字节为合法用户名。求线性方程组的核并筛选候选，可得到一个 128 位密钥，使原 `ct` 解密为 `Send flag to <target>`。

最后完成消息替换：

1. 注册可控的 `<target>` 用户并保存其私钥。
2. 预测随机数，登录 `admin`。
3. 保持原欢迎邮件的 `ct` 和签名不变。
4. 用 `flag_haver` 的公开密钥重新封装刚求出的流密码密钥。
5. 以 `admin` 身份把组合后的消息发给 `flag_haver`。
6. Bot 验证旧签名仍然有效，解密后执行命令，并向 `<target>` 发送 flag。
7. 用 `<target>` 私钥解开 ElGamal 密钥封装，再解密流密码正文。

最终消息为：

```text
DUCTF{wait_its_all_linear_algebra?...always_has_been}
```

## 方法总结

这是一条跨原语组合漏洞链。MT19937 的线性状态泄漏破坏身份认证；自定义流密码允许求任意目标明文对应的密钥；签名只覆盖密文而不覆盖收件人和封装密钥，允许重绑定。单个原语看似各司其职，但缺少域绑定和使用安全随机源会让整体协议完全失效。
