# 猫咪小测

## 题目简述

题目包含四个信息检索问题，分别涉及中国科学技术大学图书馆馆藏、arXiv 论文、Linux 内核配置和计算机科学论文的正式发表信息。四项答案依次为 `12`、`23`、`CONFIG_TCP_CONG_BBR` 和 `ECOOP 2023`。

## 解题过程

### 图书馆楼层

在[中国科学技术大学图书馆](https://lib.ustc.edu.cn/)检索书名时，不要把 `2nd ed.` 当成书名主体；去掉版本说明后搜索 `A Classical Introduction To Modern Number Theory`，可定位到馆藏地“西区外文书库”。馆藏界面的悬浮说明以及[西区图书馆简介](https://lib.ustc.edu.cn/%E6%9C%AC%E9%A6%86%E6%A6%82%E5%86%B5/%E5%9B%BE%E4%B9%A6%E9%A6%86%E6%A6%82%E5%86%B5%E5%85%B6%E4%BB%96%E6%96%87%E6%A1%A3/%E8%A5%BF%E5%8C%BA%E5%9B%BE%E4%B9%A6%E9%A6%86%E7%AE%80%E4%BB%8B/)都表明外文书库位于西区图书馆 `12` 楼。

### 鸡的密度上限

把题面改写成英文关键词 `arxiv observable universe chicken density`，可找到讨论可观测宇宙中鸡密度上限的预印本。摘要直接给出最严格的上限为 $10^{23}\,\mathrm{pc}^{-3}$，题目只要求指数，所以答案是 `23`。

### BBR 内核选项

以 `kernel config bbr` 检索可找到内核配置索引中的 [BBR TCP 条目](https://cateee.net/lkddb/web-lkddb/TCP_CONG_BBR.html)，其配置符号就是 `CONFIG_TCP_CONG_BBR`。若希望从内核源码独立核对，可以在任意较新的 Linux 内核源码目录运行：

```bash
make menuconfig
```

在配置界面按 `/` 搜索 `bbr`。结果中一项用于选择默认拥塞控制算法，另一项才是启用 BBR 实现；后者的完整配置名为 `CONFIG_TCP_CONG_BBR`。

### 论文发表会议

用 `python type check mypy halting problem` 等关键词可定位论文 [Python Type Hints Are Turing Complete](https://arxiv.org/abs/2208.14755)。arXiv 页面解决的是“哪篇论文”，而题目问的是“发表在哪个会议”，所以还要按作者和标题在 DBLP 或论文出版信息中交叉核对。正式发表记录指向 `ECOOP 2023`。

## 方法总结

本题的关键工作全部是从公开来源定位并交叉验证事实，因此归类为 OSINT。检索时应先把中文题面提炼为英文实体与技术关键词，再区分“搜索线索”和“最终证据”：馆藏位置要落到馆方信息，数值要落到论文摘要，配置项要落到内核配置记录，会议名则要落到正式出版记录。这样即使搜索排序变化，也不会把搜索结果摘要误当作答案本身。
