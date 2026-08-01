# GlacierCTF 2025 Secret Sharing Platform

## 题目简述

题目是一个带注册、登录、TOTP 和秘密分享功能的 Flask 平台。表面漏洞是 `/share` 只按数字 ID 读取 `Secret`，没有检查秘密是否属于当前用户；但仅凭 IDOR 只能得到管理员当前的 TOTP，仍缺少管理员密码。

决定性缺陷位于自定义 PRNG：Dockerfile 将同一个 `SSP_RNG_SEED` 固化进镜像，所有按该镜像启动的实例都使用同一条 HMAC-SHA256 输出流。攻击者可以在多个实例上以不同操作序列暴露相互重叠的流片段，拼回管理员密码所在的区间，再与 IDOR 组合接管管理员账户。

## 解题过程

### 1. 还原 TokenPRNG 的消费模型

`TokenPRNG` 以 20 字节 seed 为 HMAC key，以 16 字节大端 counter 为消息：

```python
block = hmac.new(seed, counter.to_bytes(16, "big"), hashlib.sha256).digest()
counter += 1
```

`rand_pk()` 消耗 8 字节生成对象 ID；`token_urlsafe()` 取得所需字节后做 URL-safe Base64，并会丢弃当前调用没有使用的尾部字节。因此不能把网页里见到的 token 简单首尾相接，必须按每个 API 的实际取数长度、Base64 编码和舍弃规则维护“流偏移”。

管理员初始化流程也使用这条流：创建 flag 对应的 Secret ID，并通过 `/api/generate_password` 生成新密码。应用生成 256 个原始随机字节，Base64 编码后截取前 32 个字符作为管理员密码。也就是说，密码并非不可预测，只是位于同一确定流的更早位置。

### 2. 跨实例收集并合并重叠片段

在任意新实例注册普通用户，创建自己的秘密并反复生成 share token，可以观察到以下类型的 PRNG 输出：

- 自己的用户或秘密 ID，可恢复对应的 8 字节片段；
- URL-safe Base64 的分享 token，可逆回连续原始字节；
- 不同初始化和请求顺序造成的已知消费间隔。

所有实例种子相同，但可以在不同实例里插入不同数量的操作，让可见窗口相对整条流前后平移。官方 solver 为每段数据记录全局偏移，遇到重叠区先验证字节完全一致，再执行 merge。这样逐步扩大连续已知区间，而不是依赖一次实例泄漏完整状态。

随后在当前目标实例再生成一个 token，在已拼接的长字节串中搜索该 token 的原始字节。唯一匹配位置给出了当前实例的 counter 对齐点；按源码中的初始化调用顺序向前回退，就能定位管理员密码所用原始字节和管理员 Secret ID。

### 3. 用 IDOR 取得 TOTP 并登录

`/share` 虽要求登录，却只有如下语义：按用户提交的 ID 查询秘密并创建分享链接，没有所有权判断。使用恢复出的管理员 Secret ID 创建分享，再访问分享链接，即可看到该 TOTP secret 的当前一次性验证码。

接下来使用恢复出的 32 字符管理员密码和当前 TOTP 登录管理员账户。管理员的 `/export` 接口导出保存 flag 的 Base32 secret，最后做 Base32 解码：

```python
flag = base64.b32decode(exported_secret)
```

源码实例的结果为：

```text
gctf{STr3^m_$huff1e_M3rg3_$p|1c3_r1n53_4nd-R3p34t}
```

## 方法总结

本题不是单一 IDOR：IDOR 只能越权读取 TOTP，固定种子的跨实例 PRNG 才负责恢复管理员 ID 和密码。分析此类问题时，应把每个随机 API 视为“从统一字节流消费多少、编码后暴露多少、丢弃多少”的状态机，并用重叠片段做一致性校验。最终利用链是“跨实例流重建 → 定位管理员凭据 → IDOR 获取当前 OTP → 管理员导出”，缺少任何一环都无法得到 flag。
