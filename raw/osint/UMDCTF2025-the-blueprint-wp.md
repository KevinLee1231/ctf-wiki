# UMDCTF 2025 - the-blueprint

## 题目简述

题目给出一张没有明显路牌的住宅区全景，要求确定所在街道。

![Bryn Du Drive 环绕 Alligator Mound 的全景：分叉道路与中央草丘轮廓是定位依据](UMDCTF2025-the-blueprint-wp/alligator-mound.jpg)

## 解题过程

图片没有 GPS 或描述元数据。房屋和植被都很普通，最有区分度的并非右侧临时
`ROAD CLOSED` 标牌，而是道路几何：道路分成两支，围绕中央一座人工感很强的长草丘
形成环形路线。

以“俄亥俄、道路环绕、effigy mound”等特征检索历史地点，可以找到 Granville 的
Alligator Mound。Licking County Historical Society 的
[历史地点说明](https://lchsohio.org/historic-sites/)指出，这是一座约 200 英尺宽、
最高约 6 英尺的史前动物形土丘，位置就在 Bryn Du Drive。

再用地图和街景核对：

- Bryn Du Drive 确实围绕土丘形成环形道路；
- 中央草坡的轮廓和树木位置与题图一致；
- 周围独立住宅、路缘和两侧分叉关系也一致。

因此拍摄道路的完整地址为：

```text
Bryn Du Dr, Granville, OH 43023
```

最终提交：

```text
UMDCTF{Bryn Du Dr, Granville, OH 43023}
```

## 方法总结

无文字街景应优先分析道路和地形等难以复制的结构性线索。这里的草丘不是普通绿化岛，
而是被道路环绕的历史遗址；识别出 Alligator Mound 后，公开历史资料会直接把地点
关联到 Bryn Du Drive。临时施工牌只能说明拍摄时状态，不能单独用于定位。
