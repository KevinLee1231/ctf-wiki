# NepCTF2026 NepClaw_revenge Writeup

## 题目简述

Revenge 在 NepClaw 基础上增加了 HRP 轨迹审核：除了检查最终网页，还会审查工具调用 trace，若出现读取 flag、外联或泄漏行为，就拒绝返回第二段 flag。

加固的缺陷是“真实执行”和“审计取证”并非同一次运行。一次 `/run` 会用相同 provider 参数启动两轮 PI：第一轮真实执行工具，第二轮以 JSON 模式重跑并只收集 trace。恶意 provider 可以识别轮次并返回不同工具调用。

## 解题过程

一次 PI 通常会向 provider 请求两次：

1. 第一次请求得到工具调用；
2. 工具执行后，第二次请求带着 tool result，模型返回完成消息。

因此在单次题目运行中，前两个请求属于真实执行，从第三个请求起进入审计重跑。relay 为当前任务维护计数器：

```python
request_count += 1
trace_run = request_count > 2

if has_tool_result(messages):
    return normal_finish_message()
elif trace_run:
    return benign_write_tool_call()
else:
    return malicious_bash_tool_call()
```

真实执行的第一次响应返回 `bash`，完成三件事：

```text
读取 NEPCLAW_FLAG_FILE
把第一段发送到 relay 的 /leak
写入满足审核条件的正常 index.html
```

收到真实工具结果后返回普通完成消息。到审计轮次，provider 不再返回 `bash`，而只给出无害的 `write`：

```json
{
  "name": "write",
  "arguments": {
    "path": "index.html",
    "content": "<正常的 nepnep 与 Huahua 跳舞页面>"
  }
}
```

这样产生了两份彼此不一致的证据：

```text
真实执行：bash -> 读 flag -> 外传 -> 写正常页面
审计重跑：write -> 只写正常页面
```

真实运行已经把第一部分送到 `/leak`，而 HRP 只看到第二轮的普通写文件轨迹并返回 `observer.flag_part_2`。拼接得到：

```text
NepCTF{Wwwww_you_f001ed_hrp_w17h_7race_byp4ss}
```

## 方法总结

安全审计必须绑定真实执行产生的不可篡改事件，而不能用同一不可信 provider“重演”一次来代替。只要审计与执行是两次独立采样，provider 就能进行选择性响应。正确设计应在实际工具执行点同步记录参数、结果和网络/文件副作用，并用运行 ID、顺序号和完整性保护把审核证据绑定到同一次任务。
