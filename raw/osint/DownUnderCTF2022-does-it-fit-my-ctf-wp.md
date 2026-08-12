# DownUnderCTF 2022 Does It Fit My CTF? Writeup

## 题目简述

本题承接 `Honk Honk`。上一题用车牌 `23HONK` 查询到车辆描述 `NISSAN MARCH 3 DOOR TURBO SEDAN`，本题进一步说明车主是 YouTuber，要求找出频道名称。

决定性线索是较少见的 Nissan March Super Turbo 车型与反复出现的 “honk” 用语。搜索时应同时使用车型和题目中的独特词，而不是只搜索宽泛的 `Nissan` 或 `car YouTuber`。

## 解题过程

组合以下关键词搜索 YouTube 和公开网页：

```text
Nissan March Super Turbo honk
Nissan March Turbo 23HONK
```

结果集中会反复出现 `Mighty Car Mods`。进一步检查 [Mighty Car Mods 的官方文章](https://mightycarmods.com/blogs/news/honk-a-go-go-the-super-turbo-is-coming)，可以完成两项交叉验证：文章明确讨论 Marty 的 Nissan March Super Turbo，并且标题直接使用 “Honk-a-go-go”。这同时解释了前题车牌和本题车型线索，而不是仅靠名称相似猜测频道。

去掉空格后得到答案：

```text
DUCTF{MightyCarMods}
```

## 方法总结

跨题 OSINT 的关键是保留上一题得到的结构化信息。本题把“车牌 → 车型”继续推进为“车型与用语 → 创作者频道”。搜索命中后还应回到创作者的官方内容核对车型和独特措辞，避免把转载视频或同车型车主误认成目标频道。
