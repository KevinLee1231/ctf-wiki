# TSGCTF2024 CONPASS

## 题目简述

服务端模拟四颗卫星。每个卫星端点返回一段包含发送时间的 JSON、对应 RSA 签名和公钥。`/auth` 先验证四份签名，再根据“当前时间减发送时间”估算到卫星的距离；只有四个距离都与 flag 所在原点相符时才返回 flag。

签名没有哈希和填充，直接对 JSON 的 little-endian 整数做 textbook RSA：

```python
data_int = int.from_bytes(data.encode(), 'little')
signature = pow(data_int, d, n)
```

验证只检查：

```python
data_int % n == pow(signature, e, n)
```

此外，自定义解码器会忽略非法 UTF-8，并删除所有非 printable 字符。目标是在不获取私钥的情况下，为原签名构造一个模 $N$ 同余、但过滤后成为新时间 JSON 的消息。

## 解题过程

### 1. 计算原点位置应有的发送时间

卫星 $i$ 的正常接口返回：

$$t_i=\operatorname{now}-\operatorname{dist}(sat_i,user)$$

选择一个略晚于当前时间的统一认证时刻 $T$，若用户位于 flag 原点，则四份数据中的发送时间应改为：

$$t'_i=T-\operatorname{dist}(sat_i,flag)$$

solver 用 sat0 的返回时间反推出服务器当前时间，再加 5 秒作为 $T$，从而减少本地与服务器时钟差影响。

### 2. 为同一 RSA 剩余类构造新 JSON

设原始签名消息的 little-endian 整数为 $D$，模数为 $N$。希望新消息以前缀开头：

```json
{"time": 1733460000, "d": "
```

设该固定前缀长度为 $l$ 字节，$M=256^l$，其 little-endian 低位整数为 $T_0$。需要同时满足：

$$X\equiv D\pmod N$$

$$X\equiv T_0\pmod M$$

RSA 模数为奇数，所以 $\gcd(N,M)=1$。令 $X=D+Nt$，则：

$$t\equiv(T_0-D)N^{-1}\pmod M$$

代码可写为：

```python
M = 1 << (8 * l)
t0 = (pow(N, -1, M) * (T0 - D)) % M
X0 = D + N * t0
```

所有解为：

$$X=X_0+kNM$$

改变 $k$ 不会破坏 JSON 前缀，也不会改变消息模 $N$ 的值，因此原签名仍可通过。

### 3. 选择合法结尾并利用过滤器

消息按 little-endian 解释，所以整数最高两个有效字节必须依次对应 JSON 末尾的 `"}`。solver 对若干候选总长度 $L$ 计算 $k$ 的上下界，使：

$$
\operatorname{ord}('}')256^L+
\operatorname{ord}('"')256^{L-1}
\le X <
\operatorname{ord}('}')256^L+
(\operatorname{ord}('"')+1)256^{L-1}
$$

对范围中的候选转为 little-endian 字节，并拒绝填充区含 `"` 或反斜杠的情况，避免提前结束字符串或改变 JSON 转义语义。中间的随机二进制在：

```python
bytes.fromhex(data).decode('utf-8', errors='ignore')
```

以及 printable 过滤后大多消失，剩余文本成为合法的：

```json
{"time": <forged_time>, "d": "<harmless printable filler>"}
```

但它对应的整数仍满足 $X\equiv D\pmod N$，所以复用原签名即可。

### 4. 同步提交四份伪造数据

对 sat0 到 sat3 分别使用各自公钥模数构造碰撞消息，保留原 `sign` 字段。在选定的 $T$ 附近提交 `/auth`。服务器解析出的四个时间都指向原点位置，并满足允许的 `-1` 到 20 秒误差窗口，返回：

```text
TSGCTF{I_|^_|a^^3d_i+_t|-|3_CONfi|)e|^_|(e_P05iti0|^_|_Au+|-|e|^_|+ifi(4tion_S'/5te^^_8u+_it_|^/ill_|^_|e|/3|2_5ee_the_ligh+_0f_day}
```

## 方法总结

本题把 textbook RSA 的“只认证模 $N$ 剩余类”和宽松文本规范化组合成签名绕过。CRT 让攻击者同时固定 little-endian JSON 前缀并保持与原消息模 $N$ 同余，非 printable 过滤又把用于满足同余式的二进制垃圾从解析视图中删除。安全签名必须先对唯一、规范化的字节表示做抗碰撞哈希，再使用 RSA-PSS 等标准方案；验证的字节串必须与后续解析的字节串完全一致，不能在验签后再做有损解码或过滤。
