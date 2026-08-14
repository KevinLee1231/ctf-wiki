# CakeCTF 2022 kiwi Writeup

## 题目简述

服务接收一段十六进制数据，并用 Kiwi 的 schema 解码为 `EncryptionKey`：

```text
message EncryptionKey {
  uint magic = 1;
  byte[] key = 2;
}
```

解码结果必须满足 `magic == 0xCAFEC4F3` 且密钥长度至少为 8。随后服务按下面的方式加密 flag：

$$
C_i=P_i\oplus key_{i\bmod |key|}\oplus 0xff\oplus i.
$$

目标不是攻击密码算法，而是还原 Kiwi 消息编码，构造一个内容完全已知的合法密钥。

## 解题过程

生成的 `EncryptionKey::decode` 先读取字段编号，再按字段类型读值。消息格式为：

```text
字段 1 编号 | magic 的 varuint
字段 2 编号 | 数组长度 varuint | key 字节
字段 0 结束标记
```

Kiwi 的无符号整数使用每组 7 位、低组在前的 varuint。`0xCAFEC4F3` 编码为：

```text
f3 89 fb d7 0c
```

选择 8 字节全零密钥，完整消息为：

```text
01 f3 89 fb d7 0c 02 08 00 00 00 00 00 00 00 00 00
```

各部分含义如下：

```text
01                    字段 1：magic
f3 89 fb d7 0c        0xCAFEC4F3 的 varuint
02                    字段 2：key
08                    数组长度 8
00 00 00 00 00 00 00 00
                      八字节零密钥
00                    消息结束
```

提交时去掉空格：

```text
01f389fbd70c0208000000000000000000
```

由于密钥全为零，加密式退化为：

$$
P_i=C_i\oplus0xff\oplus i.
$$

对应 solver：

```python
from ptrlib import Socket

sock = Socket("misc.2022.cakectf.com", 10044)
payload = "01f389fbd70c0208000000000000000000"
sock.sendlineafter("Enter key: ", payload)

cipher = bytes.fromhex(
    sock.recvlineafter("Encrypted flag: ").decode().strip()
)
flag = bytes(value ^ 0xff ^ i for i, value in enumerate(cipher))
print(flag.decode())
```

最终得到：

```text
CakeCTF{w3_n33d_t0_pr3v3nt_Google_fr0m_st4nd4rd1z1ng_ev3ryth1ng}
```

## 方法总结

这题的主要工作是从生成代码中恢复一个最小合法序列化消息。通过主动选择全零密钥，可以把未知输入从加密式中完全消掉，服务返回值就能逐字节逆转。

分析自定义协议时，应分别确认字段编号、整数编码、数组长度和消息终止符。少一个末尾 `00` 会让解码器继续读取字段编号并失败；这也是构造 payload 时最容易遗漏的细节。
