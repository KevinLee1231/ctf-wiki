# BYUCTF 2023 - Legoclones 3

## 题目简述

题目要求 Clone Trooper Wiki 创建时间的分钟数。普通站史只给日期，必须找到平台自身的精确页面信息。

## 解题过程

Fandom 页面支持 `action=info`，可查看页面/社区的创建元数据。访问：

[Clone Trooper Wiki 页面信息](https://clonetrooper.fandom.com/wiki/Clone_Trooper_Wiki?action=info)

记录显示创建时间为 `11:20`。题目明确忽略时区并只取分钟，所以答案为：

```text
byuctf{20}
```

## 方法总结

搜索摘要和二手站史常丢失时分秒。需要精确时间时，应优先使用平台生成的 `action=info`、修订历史或 API 元数据；同时严格按题目要求提取分钟，而不要自行换算时区。
