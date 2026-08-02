# pseudo-secure

## 题目简述

服务为每个用户调用 Python 全局 `random.getrandbits(8 * len(username))` 生成随机整数，再依次与用户名比特异或、左移 3 位、异或常量 `0x5A` 并 Base64 编码为登录密钥。变换完全可逆，攻击者可以注册任意长度的用户名并从返回密钥恢复 Mersenne Twister 的原始输出。三个管理员在攻击者注册前已经创建，各自的消息保存三分之一 flag。

## 解题过程

先逆转 Base64、常量异或、右移和用户名异或，即可从一个 16 字节用户名的密钥恢复一次 `getrandbits(128)` 输出。CPython 的 MT19937 状态由 624 个 32 位输出决定，所以注册 156 个长度为 16 的用户名，拆出 $156\times4=624$ 个 32 位字并提交给 `RandCrack`。

管理员用户名长度为 8，因此每个管理员创建时消耗 64 位，也就是两个 32 位输出。三个管理员一共比观测序列早 6 个输出。克隆状态后先回退观测到的 624 个字，再回退管理员消耗的 6 个字，便能按创建顺序预测三个管理员的随机值。

关键逆变换和伪造逻辑如下：

```python
import base64

def recover_random(encoded_key: str, username: str) -> int:
    bits = 8 * len(username)
    raw = int.from_bytes(base64.b64decode(encoded_key), "big")
    shifted = (raw ^ 0x5A) & ((1 << (bits + 3)) - 1)
    mixed = shifted >> 3
    username_int = int.from_bytes(username.encode(), "big")
    return mixed ^ username_int

def make_key(username: str, random_value: int) -> str:
    bits = 8 * len(username)
    mixed = random_value ^ int.from_bytes(username.encode(), "big")
    shifted = ((mixed << 3) & ((1 << (bits + 3)) - 1)) ^ 0x5A
    size = (shifted.bit_length() + 7) // 8
    return base64.b64encode(shifted.to_bytes(size, "big")).decode()
```

完整交互的状态恢复顺序是：

```python
from randcrack import RandCrack

rc = RandCrack()

# 对 156 个不同的 16 字节用户名执行注册；每次从返回 key 恢复 rand128。
for rand128 in recovered_registration_values:
    rc.submit(rand128 & 0xFFFFFFFF)
    rc.submit((rand128 >> 32) & 0xFFFFFFFF)
    rc.submit((rand128 >> 64) & 0xFFFFFFFF)
    rc.submit((rand128 >> 96) & 0xFFFFFFFF)

rc.offset(-624)  # 回到 156 次注册之前
rc.offset(-6)    # 再回到三个 Admin 用户创建之前

for index in range(1, 4):
    username = f"Admin00{index}"
    predicted = rc.predict_getrandbits(64)
    forged_key = make_key(username, predicted)
    # 依次登录三个管理员并读取消息，拼接即为完整 flag。
```

按官方交互脚本拼接三段管理员消息可得：

```text
tjctf{1_gu3ss_h1nds1ght_15_20/20}
```

## 方法总结

- 核心技巧：从可逆输出收集 624 个 MT19937 状态字，克隆并向前或向后定位 PRNG 状态。
- 识别信号：Python `random` 被用于认证密钥，同时攻击者能大量请求由同一全局 PRNG 生成的可逆输出。
- 复用要点：必须按 CPython `getrandbits` 的 32 位字顺序提交状态，并精确计算目标值与观测值之间消耗了多少个状态字；用户名长度决定一次调用消耗的字数。
