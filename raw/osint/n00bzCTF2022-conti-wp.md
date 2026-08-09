# Conti

## 题目简述

题面要求从 2022 年公开的 Conti/TrickBot 泄露资料中找到名为 `lero` 的组件凭据，答案格式为 `n00bz{username, password}`。

## 解题过程

公开报道说明 `lero` 和 `dero` 是泄露出的 TrickBot 服务端组件。沿该线索定位到 vx-underground 的 Conti 资料镜像，下载 `Conti Trickbot Leaks.7z`，用恶意样本资料常用口令 `infected` 解压。目标文件位于：

```text
lero/doc/credentials
```

其中记录的控制面板凭据为：

```text
alkahov4:nt5Sbt5ZF&$qr*T
```

按题目规定的逗号分隔格式提交：

```text
n00bz{alkahov4, nt5Sbt5ZF&$qr*T}
```

## 方法总结

本题的关键不是泛搜“Conti 密码”，而是先从报道中确认 `lero` 属于 TrickBot，再进入对应泄露包的组件文档目录。公开泄露站点和搜索结果可能失效，因此 WP 必须写清资料包名称、解压口令和内部路径，避免把解法完全寄托在外链上。
