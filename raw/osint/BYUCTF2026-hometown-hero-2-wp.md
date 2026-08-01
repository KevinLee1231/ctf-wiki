# Hometown Hero 2

## 题目简述

第二题要求找出 Cameron Snider 小学就读的学校。已知上一题的地点是 Kearney, Nebraska，可把搜索范围限制到当地历史报道。

## 解题过程

以姓名和候选学校类型组合精确搜索，例如：

    "cameron snider" "park elementary"

结果中的 Kearney Hub 地方新闻标题为 “Second-graders brighten holiday for military”。报道及配图列出 Park Elementary 的学生，并出现童年时期的 Cameron Snider；文章所在地、年龄和上一题的 Kearney 结论一致。

因此提交学校名称：

    byuctf{Park_Elementary}

这里所谓的“枚举”发生在搜索关键词层：用 Kearney 当地小学名称逐步收紧查询，而不是向 CTF 平台反复猜 flag。

## 方法总结

地方报纸、学校活动报道和图片说明常能补足早年经历。应把已确认的城市作为强约束，并用姓名的精确匹配降低噪声；最终还要核对报道时间与人物年龄，避免误中同名成年人。
