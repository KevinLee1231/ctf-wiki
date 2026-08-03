# UIUCTF 2023 Tornado Warning Writeup

## 题目简述

附件 `warning.wav` 是一段约 93 秒的单声道、44.1 kHz、16 位 PCM 天气广播录音。开头包含 Specific Area Message Encoding（SAME）报头：同一串数据连续发送三次，接收机通常会对每个字符做“三取二”纠错。

本题故意让三个报头在部分位置出现差异。天气收音机纠错后仍能恢复正常报头，而被纠掉的异常字符构成一条隐蔽消息。决定性障碍是保留三次原始解调结果并提取差错，而不是分析语音内容。

## 解题过程

首先使用能够输出未经多数投票纠错的 SAME 解调器处理音频。官方解法使用 `nwsrx`：

```bash
nwsrx warning.wav
```

应保留输出中的前三条报头。若所用工具会自动把三次发送合并成一条，就无法看到藏在差异中的字符；可以关闭其纠错逻辑，或在音频编辑器中把每次报头分别复制三遍后逐段解调。

设三条等长报头为 $H_1,H_2,H_3$。逐列比较字符：

- 正常情况下，原始 SAME 字符在三条报头中出现两次，异常字符只出现一次，取唯一字符；
- 若待隐藏字符恰好等于正常字符，则该列三个字符完全相同，也要保留该字符；
- 最后去掉两端填充，只截取 `UIUCTF{...}`，并转为小写以符合本题不区分大小写的 flag 规则。

完整提取逻辑如下：

```python
import subprocess

result = subprocess.run(
    ["nwsrx", "warning.wav"],
    capture_output=True,
    check=True,
    text=True,
)
headers = result.stdout.splitlines()[:3]

if len(headers) != 3 or len({len(header) for header in headers}) != 1:
    raise ValueError("未取得三条等长的原始 SAME 报头")

hidden = []
for column in zip(*headers):
    for character in column:
        count = column.count(character)
        if count == 1 or count == 3:
            hidden.append(character)
            break

padded = "".join(hidden)
start = padded.index("U")
end = padded.index("}", start) + 1
print(padded[start:end].lower())
```

提取结果为：

```text
uiuctf{3rd_w0rst_tor_outbre@k_ev3r}
```

## 方法总结

SAME 重复发送报头本来是为了容错，本题却把“少数派字符”改造成隐蔽信道。常规接收软件会主动消除这些差异，因此取证时要避免只保留纠错后的结果。面对重复编码载体，应同时比较原始副本和纠错输出：多数值承载正常数据，少数值、校验残差或时序差异可能承载隐藏信息。
