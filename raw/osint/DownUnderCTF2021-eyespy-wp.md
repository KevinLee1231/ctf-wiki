# DownUnderCTF 2021 - eyespy

## 题目简述

题目提供一份两页通话记录 PDF，要求根据通话日期、时间、前序地点线索和对飞机的描述，找出嫌疑人同伙所乘航班的飞机注册号及目的地。前序题已把 Isa Haxmoore 的活动地点定位到 Canberra/ACT，因此航班检索应从 Canberra Airport 出发。

## 解题过程

### PDF 通话记录转写

PDF 第 1 页标注日期 `19/09/2021`、记录号 `#064123842921`、通话时长 `7 min 19 secs`；`UM` 表示 unidentified male，`IH` 表示 Isa Haxmoore。逐行内容如下：

| 时间 | 说话人 | 内容 |
|---|---|---|
| 14:30 | UM | (inaudible) |
| 14:30 | IH | Hello? |
| 14:31 | IH | Hello?... |
| 14:31 | UM | Oh hey mate, sorry I forgot I was on mute |
| 14:31 | UM | How have you been? Settling in well since moving from SA? |
| 14:32 | IH | [REDACTED] |
| 14:35 | UM | You’ll have to show me one day |
| 14:35 | UM | Im so bored right now, how long does this guy take? |
| 14:35 | IH | I don’t know, he was supposed to meet me here 8 minutes ago. |
| 14:36 | UM | Damn his flight leaves very soon, are you sure he’ll make it? |
| 14:36 | IH | (inaudible background noise) |
| 14:37 | IH | I’ve got to bounce, he is here, I’ll call you back in a few minutes |

PDF 第 2 页标注同一日期、记录号 `#06412384292`、通话时长 `5 min 35 secs`：

| 时间 | 说话人 | 内容 |
|---|---|---|
| 15:06 | IH | Hey, just met him. I’m not sure he’ll make it in time, I think it will be close. |
| 15:06 | UM | Yeah fair dinkum, what took him so long? |
| 15:07 | IH | He said he dropped into the servo but I could have sworn I saw Macca’s in his car. |
| 15:07 | IH | Anyway, did you ever play eye spy as a kid? |
| 15:07 | UM | Yeah, I did I was never any good at it though |
| 15:07 | IH | …. (pause) Oh wait! I got one! |
| 15:08 | IH | Eyespy with my little eyes, something beginning with “P”. |
| 15:08 | UM | You know this is a phone call right, I can't see what you are looking at... |
| 15:09 | IH | Oh right yeah… Well have a go mate |
| 15:09 | UM | Yeah nah, I have no idea, lots of things start with ‘P’. Can you at least give me a fair go, what about a hint? |
| 15:09 | IH | Okay, it’s go to do with (inaudible) |
| 15:10 | UM | (pause) Is it a plane? |
| 15:10 | IH | Yeah, one that’ll touch down in about an hour |
| 15:10 | IH | Before I go, do you have a means of transport? |
| 15:11 | UM | Yeah I’ll just take the train again, that’s what I did last time. We ended up meeting under a bridge, took me forever to find it. I spent ages waiting at the bridge on Hull Road in Melbourne... |
| 15:11 | IH | [REDACTED] |
| 15:11 | IH | Alright, just wanted to confirm those details with you |
| 15:12 | UM | Yes, that sounds like a plan. As we discussed earlier, I gave the decryption key to [REDACTED] on the USB earlier, they said they would meet you at the dead drop location |
| 15:12 | IH | Catcha! |
| 15:12 | UM | Cya mate |

### 航班关联

通话给出四个决定性约束：

1. 日期是 2021 年 9 月 19 日；
2. 前序定位表明出发地在 Canberra；
3. 15:06 左右同伙刚赶到，航班即将离开，15:10 又能看到相关飞机；
4. 飞机约一小时后落地，且对话明确提到 Melbourne。

在历史航班记录中检查 Canberra 于 15:00 前后出发、飞行时间约一小时的航班，可找到一班 15:05 离开 Canberra、16:01 抵达 Melbourne 的航班，实际飞行时间 56 分钟，完全符合时间窗。该航班飞机注册号为：

```text
VH-YIB
```

因此提交：

```text
DUCTF{VH-YIB_melbourne}
```

仓库旧版 `WRITEUP.md` 的叙述处曾把注册号误写成 `YH-YIB`，但同一文件的最终 flag 和 `challenge.yml` 都是 `VH-YIB`；澳大利亚民用飞机注册前缀也应为 `VH-`，所以应采用后者。

## 方法总结

本题需要把 PDF 中的自然语言转换为结构化航班约束：出发地、日期、最早/最晚起飞时间、预计航程和目的地候选。历史航班平台的免费数据保留期很短，因此长期 WP 不能只写“去查 FlightRadar24”，而应保存完整对话、筛选条件、命中航班的起降时间、注册号和交叉验证依据。
