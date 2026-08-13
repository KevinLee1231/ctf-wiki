# Shapes Galore

## 题目简述

附件是带 VBA 的 Word 文档 `shapes.docm`，页面包含大量图形和一个“show FLAG!!!!”宏按钮。官方题解给出的完整主线是运行该宏，再到文档自定义属性中读取 `x_flag`。对照附件可知，宏会按编号收集图形描述中的片段、拼接 flag，并把结果写入该属性。

## 解题过程

应只在隔离的 Office 分析环境中打开文档并启用宏。点击“show FLAG!!!!”后，依次打开 Word 的文件属性、详细属性和自定义属性，在 `x_flag` 字段读取：

```text
grey{w0w_1_l3arnt_m4cr0s_t0d4y!!!!!!}
```

为解释该结果如何生成，可把 DOCM 作为 ZIP 容器静态查看。`word/document.xml` 的图形 `descr`/`alt` 属性包含以下片段：

```text
seg:01:gr
seg:02:ey{
seg:03:w0w
seg:04:_1_
seg:05:l3a
seg:06:rnt
seg:07:_m4
seg:08:cr0
seg:09:s_t
seg:10:0d4
seg:11:y!!
seg:12:!!!!}
```

文档还含有 `seg:99:NOT_THIS`、`decoy:...`、`note:...` 和 `grey:this_not_flag` 等干扰项。VBA 工件中的可读字符串说明宏会按索引排序合法片段、检查结果是否以 `grey{` 开头，再写入 `x_flag`。初始 DOCM 中没有 `docProps/custom.xml`；该属性是宏运行后才创建或写入的，而不是预先藏在文件属性中。

## 方法总结

- 核心技巧：运行题目宏后从 Word 自定义属性读取 `x_flag`；静态解包 DOCM 可解释宏如何从图形描述构造该值。
- 识别信号：宏按钮、`MACROBUTTON BuildFlagToProperty` 字段、图形 alt/descr 中结构化的 `seg:<index>:<data>`。
- 复用要点：未知 Office 宏只能在隔离环境中运行；需要审计时再离线解包，并依据严格格式和索引范围排除干扰图形，不要按肉眼位置猜顺序。
