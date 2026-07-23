# vlad pad

## 题目简述

题目给出一处街巷的 Google Street View 全景，要求找出附近一家 IT 公司的名称。题面中的 Vladimir、Harkonnen 等 Dune 语义是公司检索的额外提示。

![Harkonnen 公司附近绿白建筑、街边小店和架空线的全景](UMDCTF2024-vlad-pad-wp/harkonnen-building-street.jpg)

## 解题过程

全景里的英文 `FOR RENT` 手写牌、密集架空线、热带植被、路边小型杂货摊和车辆环境都更接近菲律宾城市街区。先以这些特征把国家候选缩到菲律宾，再利用题名 `vlad pad` 和题面人物 Vladimir Harkonnen 提取关键词 `Harkonnen`。

搜索菲律宾境内名称包含 `Harkonnen` 的科技或 IT 企业，可以找到 `Harkonnen Industries Corporation`；[菲律宾公司注册资料](https://companieshouse.ph/harkonnen-industries-corporation) 可核对其完整注册名称。随后回到地图检查其周边街景，目标建筑的绿白外墙、底层拱形开口、右侧绿色棚顶小店、狭窄巷道和架空线位置均与附件一致，从而排除只是同名搜索结果的可能。[MapleBacon 的赛时定位记录](https://maplebacon.org/2024/05/umdctf2024-vlad-pad/) 保留了从菲律宾候选到公司周边街景的核对图，而决定结论的视觉特征已转写在本段中。

公司全名为：

```text
Harkonnen Industries Corporation
```

最终提交：

```text
UMDCTF{Harkonnen_Industries_Corporation}
```

## 方法总结

题面彩蛋可以生成高价值关键词，但必须经过地理证据确认。本题的可靠链条是“街景判断菲律宾 → Dune 主题导出 Harkonnen → 搜索公司 → 用建筑和街巷结构反向核验”，而不是看到主题词后直接猜公司名。
