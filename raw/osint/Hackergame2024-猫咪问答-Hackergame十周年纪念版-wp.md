# 猫咪问答（Hackergame 十周年纪念版）

## 题目简述

题目由六个信息检索问题组成，范围覆盖 Hackergame 历史、科大图书馆热搜、USENIX Security 2024 论文、Linux 主线提交以及 Llama 3 tokenizer。前五问的决定性证据来自公开网页、论文或 Git 仓库；第六问需要对当前题目 HTML 使用指定 tokenizer 精确计数。

## 解题过程

### 前五问：建立可追溯证据链

1. 在 [中国科大第二届信息安全大赛存档页](https://lug.ustc.edu.cn/wiki/sec/contest.html) 中查到 Hackergame 2015 赛前讲座地点为 `3A204`。
2. 先统计 2019—2023 各届题目数，2019 年的 28 题最接近题面所说的约 25 题；再查 [Hackergame 2019 赛后总结](https://lug.ustc.edu.cn/news/2019/12/hackergame-2019/)，注册人数为 `2682`。
3. Hackergame 2018 的相关记录指出，猫咪问答让科大图书馆当月热搜第一变成 `程序员的自我修养`。这里需要提交的是检索词，不是完整书名或索书号。
4. 论文 *FakeBehalf: Imperceptible Email Spoofing Attacks against the Delegation Mechanism in Email Systems* 提出 6 类攻击，并测试 16 个邮件服务、20 个客户端以及服务商 Web 界面，共 `336` 种组合。关键计算是 `16 × 20 + 16 = 336`，论文原文也直接给出该数值。
5. 从 Greg Kroah-Hartman 于 10 月 18 日提交的 MAINTAINERS 清理 patch 沿提交历史追到 Linux mainline，完整 commit ID 为 `6e90b675cf942e50c70e8394dfb5862975c3b3b2`；题目接受其唯一前缀 `6e90b6`。

### 第六问：按指定模型精确计数

必须使用题目指定的 `meta-llama/Meta-Llama-3-70B` tokenizer 和带会话状态的题目 HTML；不能用另一模型的在线 API 近似。复现代码如下：

```python
import requests
import transformers

tokenizer = transformers.AutoTokenizer.from_pretrained(
    "meta-llama/Meta-Llama-3-70B"
)
session = requests.Session()
html = session.get(
    "<题目页面>",
    cookies={"session": "<自己的会话 Cookie>"},
).text

# encode() 默认加入一个 BOS token，它不是 HTML 正文的一部分。
answer = len(tokenizer.encode(html)) - 1
print(answer)
```

结果为 `1833`。最终六项答案依次为：

```text
3A204
2682
程序员的自我修养
336
6e90b6
1833
```

## 方法总结

- 核心技巧：把每一问拆成“定位权威来源—提取精确字段—按题目格式归一化”，并对模型相关问题使用指定版本本地复现。
- 识别信号：题目给出年份、人物、论文主题、patch 日期和模型全名，这些都是缩小检索范围的稳定锚点。
- 复用要点：优先使用比赛存档、论文原文和上游 Git 提交；人数、组合数、commit 前缀与 token 数都要区分“来源原值”和“提交格式”。tokenizer 计数还要确认是否自动添加 BOS/EOS 等特殊 token。
