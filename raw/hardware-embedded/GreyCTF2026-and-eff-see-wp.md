# And Eff See (do you see?)

## 题目简述

现场提供一张只允许读取的 NTAG213（NFC Forum Type 2）卡。普通读取能看到 7 字节 UID 和提示 `xor uid halves!!`，但从第 8 页开始的内容受 32 位密码保护。目标是在不写卡的前提下推导每张卡自己的密码，认证后读出明文 flag。

NTAG213 使用 4 字节页和 `PWD_AUTH`，不是 MIFARE Classic 的 Crypto1/扇区密钥体系，因此不能套用 Classic 的认证命令。

## 解题过程

先用支持 NTAG 原始信息的 NFC 手机应用或 USB 读卡器读取公开页。示例卡的 UID 为：

```text
04 A1 B2 C3 D4 E5 F6
```

公开数据页还会拼出：

```text
xor uid halves!!
```

配置把 `AUTH0` 设为 `0x08`，所以第 8 页及之后的读取需要认证。提示中的“两半”是两个相互重叠的 4 字节切片：`UID[0:4]` 与 `UID[3:7]`。逐字节 XOR 得到密码：

$$
PWD_i=UID_i\oplus UID_{i+3},\qquad 0\le i<4
$$

对上述示例：

```text
front = 04 A1 B2 C3
back  = C3 D4 E5 F6
PWD   = C7 75 57 35
```

也可直接计算：

```python
uid = bytes.fromhex("04A1B2C3D4E5F6")
pwd = bytes(uid[i] ^ uid[i + 3] for i in range(4))
```

注意示例密码不能用于其他卡；每张卡必须使用自身 UID 重新计算。向卡发送只读认证命令：

```text
1B || PWD
```

即 NTAG 的 `PWD_AUTH`。收到 PACK 表示认证成功。随后从第 8 页开始发送 `30 <page>`；一次 `READ` 返回连续 4 页、共 16 字节。依次读取 `0x08, 0x0c, 0x10, ...` 并按 ASCII 拼接，得到：

```text
grey{0h_i_s33_y0u_4re_qu1te_g00d_at_th1s_nfc_th1n9_HuH}
```

整个过程只执行识别、认证和读取，没有写页、改密码或修改配置，符合题目的只读限制。

## 方法总结

先识别标签家族比盲试工具命令更重要：NTAG213 的核心边界是页级 `PWD_AUTH`。32 位密码本身不适合现场爆破，但公开页已经泄露了自定义派生规则。UID 两个重叠切片的 XOR 把每卡唯一标识变成认证密码；认证只解除读保护，不需要也不应修改卡片内容。
