# UMDCTF 2018 - Threat Memetelligence

## 题目简述

题目只给出域名 `chall.ctf.science`，要求找到与它关联的恶意样本。这是一道威胁情报关联题：需要从域名指标反查沙箱报告，再从报告和样本字符串中提取 flag。

## 解题过程

以域名作为 IOC 在公开恶意样本分析记录中搜索，可以定位到 [Hybrid Analysis 报告](https://www.hybrid-analysis.com/sample/0d48898b7cbfa8831adc59ab9e4ca7a4dfaa924289b405f31b7050b0b33cc11e/5ad17e607ca3e1386f5b4109)。报告对应样本的关键信息为：

```text
SHA-256: 0d48898b7cbfa8831adc59ab9e4ca7a4dfaa924289b405f31b7050b0b33cc11e
Name:    payload.doc
Size:    35328 bytes
```

网络行为还显示样本向以下目标发送 HTTP POST：

```text
159.89.239.89:80
Path: /bitconnect
```

这些域名、样本哈希、文件名和网络行为共同完成了 IOC 关联，不需要依赖页面上的风险评分。报告的字符串结果中可以直接恢复：

```text
UMDCTF-{hybrid_analytica}
```

外链保留是为了指向原始沙箱证据；即使报告失效，完成解题所需的样本标识、网络指标和 flag 已在本文中完整记录。

## 方法总结

威胁情报题应形成可复核的证据链：初始域名 $\rightarrow$ 样本报告 $\rightarrow$ 文件哈希与网络行为 $\rightarrow$ flag。只写“在某网站搜到”无法解释关联依据，也会让 WP 完全依赖外部页面继续存在。
