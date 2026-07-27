---
type: technique
tags: [web, sqli, blind-sqli, filter-bypass, oracle]
skills: [ctf-web]
raw:
  - ../raw/web/sqli-filter-and-oracle.md
updated: 2026-07-27
---

# SQLi Filter and Response-Oracle Extraction

## 适用场景

用户输入进入 SQL 语句，但输出受过滤、无回显或只暴露布尔/timing/error 差异；目标是先闭合 SQL 上下文，再建立可靠 oracle 逐步恢复数据。

## 识别信号

- 引号、括号、排序、重复参数或类型变化会改变查询结果。
- 响应长度、状态、错误或延迟可区分真假谓词。
- WAF/黑名单阻止关键字、空格、逗号或注释。

## 最小证据

- 一对只差真假条件的请求产生稳定可分类响应。
- 明确数据库方言、注入上下文和可用运算符。
- 量化过滤发生在 URL、框架、WAF 还是数据库前。

## 解法骨架

1. 用最短 payload 确认字符串/数值/标识符/排序上下文。
2. 枚举同义关键字、编码、注释、空白和表达式替代，找稳定语法。
3. 建立布尔、错误或 timing oracle，优先按长度/字符范围二分。
4. 保存进度和重试统计，恢复后用独立查询或业务行为验证。

## 关键变体

- Boolean blind：页面内容或行数区分谓词。
- Time blind：使用条件延迟并做重复统计。
- Error/order oracle：错误文本、排序位置或类型转换泄露数据。

## 常见陷阱

- 把缓存、网络抖动或动态页面误当 oracle。
- 过滤绕过成功但 SQL 上下文未闭合。
- 逐字符线性枚举，忽略二分和批量编码。

## 关联技巧

- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)
- [adaptive-oracle-response-modeling.md](adaptive-oracle-response-modeling.md)
- [request-view-normalization-differentials.md](request-view-normalization-differentials.md)

## 原始资料

- [sqli-filter-and-oracle.md](../raw/web/sqli-filter-and-oracle.md)
