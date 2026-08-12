# DownUnderCTF 2021 - Heart of the nation

## 题目简述

题目提供一张林地照片，并给出 “Right at the heart of the nation, no piece of the bush inside the circle remains untouched by us” 的说明，要求定位到三位小数。照片中可见土路、半圆形石砌设施、近处灌木以及后方路灯，说明地点虽然处于林地，但紧邻城市道路。

![林地土路尽头的半圆形石砌设施与后方路灯构成主要定位特征](DownUnderCTF2021-heart-of-the-nation-wp/stone-circle-canberra.jpg)

## 解题过程

“heart of the nation” 的字面解释会先想到澳大利亚地理中心或 Alice Springs，但照片中的茂密林木和城市路灯与典型内陆环境不符。另一种解释是国家政治中心 Canberra；它又长期被称为 “Bush Capital”，同时解释了 `nation` 与 `bush` 两组提示。

进一步把 `inside the circle` 当作道路提示。Canberra 的 Parliament House 周围有 Capital Circle 与 State Circle，正好形成环绕联邦政府核心的道路系统。于是把地图搜索范围缩到 State Circle 内侧，而不是遍历整个 ACT。

在卫星图、街景和 Photo Sphere 中比较以下特征：

- 半圆形石墙的形状与开口方向；
- 土路相对树线的走向；
- 石墙后方路灯的数量、间距与对齐关系；
- 林地边缘与 State Circle 道路的相对位置。

匹配位置约为：

```text
-35.306943, 149.120724
```

仓库验题配置使用的标准答案是：

```text
DUCTF{-35.306,149.120}
```

配置也接受经纬度对调和逗号后带空格的形式。需要特别注意：官方题解给出的六位坐标若按数学规则四舍五入会得到 `-35.307,149.121`，而实际判题值采用的是截取后的 `-35.306,149.120`，复现时应以验题配置为准。

## 方法总结

这是一条“语言提示缩小区域，再用视觉几何精确匹配”的地理定位链。先用植被和城市设施排除纯字面地点，再把 `heart`、`bush`、`circle` 分别映射到 Canberra、Bush Capital 和 State Circle；最后用石墙、路灯与道路方向完成确认。坐标题还要区分纬经顺序、截断与四舍五入，并以实际接受格式校验结果。
