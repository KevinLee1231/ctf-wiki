# BYUCTF 2022 - Oh the Vanity

## 题目简述

题面称当月有文章介绍一种“掩盖钓鱼活动”的新方法，并给出鱼戴鲨鱼面具、鲨鱼戴鱼面具的配图。目标是找出文章发布日期，格式为 `mm-dd-yyyy`。

![鱼与鲨鱼互戴面具的文章配图](./BYUCTF2022_Oh_the_Vanity/masked-shark-fish-clue.webp)

## 解题过程

把题面措辞中的 “mask phishing campaigns” 与图片主题组合搜索，再加入 “vanity” 限定，可定位 Dark Reading 文章 [Vanity URLs Could Be Spoofed for Social Engineering Attacks](https://www.darkreading.com/cloud-security/vanity-urls-could-be-spoofed-for-social-engineering-attacks)。

文章的关键内容是：攻击者可以注册或利用带有可信品牌外观的 vanity 子域名/短链接，使 Box、Google、Zoom 等服务上的钓鱼落地页看起来更可信，从而在社工攻击中掩盖真实意图。页面使用的正是题目所给“鱼/鲨鱼面具”配图，排除了同日转载或主题相似文章。

文章标注发布日期为 May 11, 2022。按要求转换：

```text
byuctf{05-11-2022}
```

## 方法总结

图片与题面短语是联合指纹。只搜索“phishing”会得到大量结果；加入 `vanity URL`、文章月份和独特配图后，可同时确认文章身份与发布日期。正文已概括文章核心风险，外链仅用于来源核验。
