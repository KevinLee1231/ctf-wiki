# master of album

## 题目简述

题目是音乐专辑知识问答系统，需要在限时内答对足够多题。核心不在 Web 漏洞，而在题库/音频/封面识别自动化和稳定提交。

## 解题过程

题目入口是 `http://<题目环境地址>`，本质是一个专辑知识问答系统。每轮 40 题，限时 6 分钟，答对 30 题及以上后服务端在 `quiz_finished` 事件里返回 flag。题型包括专辑封面填名称、填艺人、选年份、四图选择，以及音频片段填专辑、填艺人、听歌选封面。

题目主要分成两类识别任务：

- 封面题：从专辑封面识别专辑名、艺人或年份，可以使用封面库、图片搜索、aHash/pHash 或带图像搜索能力的 LLM 辅助匹配。
- 音频题：从音频片段识别专辑或艺人，可以使用 Shazam、网易云接口或本地音频相似度缓存；识别结果需要处理重制版、豪华版、大小写和艺人名差异。

前端使用 Socket.IO 通信。正常流程是先 `POST /api/login`，再用返回的 `session_id` 通过 socket 发起答题。关键漏洞在 `start_quiz`：服务端没有强制校验 `team/token/session_id` 必须来自 `/api/login` 的真实登录会话。客户端可以直接构造随机队伍名、随机 token 和同值 `session_id`，然后发 `start_quiz`。随机队伍能正常收到题目、获取图片和音频、提交答案；如果某一轮答够 30 题，`quiz_finished` 返回的仍然是同一个全局真实 flag。

核心随机队伍逻辑如下：

```python
tok = secrets.token_hex(8)
team = "train_" + tok
session_id = tok
sio.emit("start_quiz", {"team": team, "token": tok, "session_id": session_id})
```

服务端返回过的结果形态如下：

```text
FINISHED {'timeout': False, 'passed': True, 'flag': 'ACTF{DouLikeMusic?
_l3tUS$1ng2GeTHEr_^o^y&h0p3_tH1s_t1mE_th3_n1tw0Rk_1$_n0Tb@D_o(T-To)}'} STATS
{'img_known': 11, 'img_total': 20, 'audio_known': 20, 'audio_total': 20, 'q':
40}
```

自动答题分两部分。图片题使用本地封面标签、RYM/iTunes 封面库、aHash/pHash 做识别；四图题根据题干提取年份或艺人，再判断对应 slot。音频题优先使用本地音频相似度缓存，未命中时调用 `ffmpeg` 转 WAV 后用 Shazam 识别，再修正常见重制版、豪华版、大小写和艺人名差异。

因为题目每轮随机抽题，且图片题库覆盖不完整，脚本通过率低。随机到图片覆盖较好的轮次才容易过；随机到大量陌生封面时会失败。最终脚本加入了图片题命中数筛选，前 20 道图片题命中不够就提前丢弃该轮，避免继续浪费时间跑音频题。

交互脚本的关键不是固定题库文件，而是绕过正式队伍次数限制后稳定跑答题流程。下面是可复用的 Socket.IO 骨架，`solve_image_question()` 和 `solve_audio_question()` 需要接入本地封面库、图片搜索、Shazam/网易云识别或人工整理出的缓存：

```python
import secrets
import socketio

BASE = "http://<题目环境地址>"
sio = socketio.Client()

state = {
    "answers": {},
    "finished": None,
}

def solve_image_question(q):
    # 根据 q 中的图片 URL、候选项和题型，返回专辑名/艺人/年份/选项。
    # 可用 aHash/pHash、本地封面库、iTunes/RYM 搜索或带图像搜索的 LLM 实现。
    raise NotImplementedError

def solve_audio_question(q):
    # 下载音频片段，必要时用 ffmpeg 转成 wav，再调用 Shazam/网易云或本地音频指纹库。
    # 返回前要做大小写、remaster/deluxe 等干扰词归一化。
    raise NotImplementedError

@sio.on("question")
def on_question(q):
    qid = q.get("id") or q.get("qid")
    qtype = q.get("type", "")
    if "audio" in qtype:
        answer = solve_audio_question(q)
    else:
        answer = solve_image_question(q)
    sio.emit("submit_answer", {"question_id": qid, "answer": answer})

@sio.on("quiz_finished")
def on_finished(data):
    state["finished"] = data
    print("FINISHED", data)
    sio.disconnect()

tok = secrets.token_hex(8)
team = "train_" + tok
session_id = tok

sio.connect(BASE, transports=["websocket"])
sio.emit("start_quiz", {
    "team": team,
    "token": tok,
    "session_id": session_id,
})
sio.wait()
```

页面提示解密成功并给出 flag，同时说明已生成 PDF 文件；后续把该 PDF/图片继续按题目逻辑处理即可。

最终答案：

```text
ACTF{DouLikeMusic?
_l3tUS$1ng2GeTHEr_^o^y&h0p3_tH1s_t1mE_th3_n1tw0Rk_1$_n0Tb@D_o(T-To)}
```

## 方法总结

- 核心技巧：把多类型专辑题转为可自动查询或本地匹配的数据集问题。
- 识别信号：题面是限时问答、题型固定、正确率阈值明确时，应优先构造题库映射和自动答题流程。
- 复用要点：分别处理封面、艺人、年份和音频片段，交互脚本要容错网络延迟和题型分支。
