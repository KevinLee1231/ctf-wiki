# 猫咪问答 Pro Max

## 题目简述

题目要求回答五个可由公开资料验证的问题，涉及已下线的网站、USTCLUG 历史、活动室照片、SIGBOVIK 论文和愚人节 RFC。关键不在“知道校内轶事”，而在于为每个答案建立可复核的来源链；已下线页面要借助网页存档或镜像。

## 解题过程

### 1. SEC@USTC 章程通过日期

原域名已经失效，但 [Internet Archive 中对 sec.ustc.edu.cn 的历史快照](https://web.archive.org/web/*/http://sec.ustc.edu.cn/) 和 USTCLUG 保存的旧站镜像仍能找到章程。章程正文明确写着：在 2015 年 5 月 4 日经会员代表大会审议通过，因此答案为：

```text
20150504
```

若存档暂时不可访问，也可在其余四题确定后按日期枚举；这只是兜底，公开存档才是证据来源。

### 2. 近五年五星级社团次数

[USTCLUG 介绍页](https://lug.ustc.edu.cn/wiki/intro/)列出了 2017、2018、2019、2020 四次，但页面更新时间早于 2021 年评选。中国科大 2021 年学生社团游园会的现场照片又显示 USTCLUG 当年仍为五星级社团，所以近五年合计：

```text
5
```

这里不能只抄旧介绍页，必须检查资料的更新时间并补上缺失年份。

### 3. 活动室门牌小字

USTCLUG 关于西区图书馆 206 活动室启用的旧文章附有门口照片；照片中 `LUG @ USTC` 下方写着：

```text
Development Team of Library
```

正文还给出了活动室位于西区图书馆二楼西北角，可用于确认照片地点而不是同名招牌。

### 4. SIGBOVIK 论文的数据集数

在 SIGBOVIK 2021 论文集里找到 *The Newcomb-Benford Law, Applied to Binary Data: An Empirical and Theoretic Analysis*。论文说明每个数据集对应一幅验证图，附录列出的图号是 2 到 14，包含端点计算为：

$$14-2+1=13$$

所以答案为 `13`。

### 5. Protocol Police 举报地址

[RFC 8962](https://www.rfc-editor.org/rfc/rfc8962.html) 第 6 节规定，所有违规报告和线索都应发送到 Unix 的黑洞设备：

```text
/dev/null
```

依次提交五个答案：

```text
20150504
5
Development Team of Library
13
/dev/null
```

服务端接受后返回账号相关的 flag。

## 方法总结

- 核心技巧：组合网页存档、官方历史页、原始照片、论文正文和 RFC，逐项建立证据链。
- 识别信号：问题指向已下线域名、带时间范围的统计或具体文档措辞时，必须检查存档时间、更新日期和原文上下文。
- 复用要点：搜索结果摘要只能定位资料，不能替代证据；区间计数要包含两端，历史页面还要防止“内容正确但尚未更新”的陷阱。
