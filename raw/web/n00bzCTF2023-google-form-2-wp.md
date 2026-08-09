# Google Form 2

## 题目简述

题面中异常大写字母拼出 `VIEW PREVIOUS RESPONSES`，提示查看 Google Form 的历史响应统计，而不是继续分析当前填写页。

## 解题过程

把表单 URL 末尾的：

```text
/viewform
```

改为：

```text
/viewanalytics
```

公开统计页会展示先前提交内容，其中包含：

```text
n00bz{7h1s_1s_th3_3nd_0f_g00gl3_f0rm5_fl4g_ch3ck3rs}
```

## 方法总结

本题利用的是表单所有者公开了响应摘要。看到题面中的大小写异常时应先提取隐藏指令，再检查同一服务的公开视图，而不是尝试绕过不存在的校验逻辑。
