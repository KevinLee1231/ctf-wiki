# UMDCTF 2025 - swag-like-ohio

## 题目简述

题目给出一张跨河公路桥全景，要求提交桥梁的地址。

![横跨 Muskingum River 的 Putnam Bridge 全景：装饰灯柱、黑色栏杆和相邻钢桁架桥用于定位](UMDCTF2025-swag-like-ohio-wp/putnam-bridge.jpg)

## 解题过程

图中可以提取出几组适合反向检索和地图核对的特征：

- 桥面为双向公路，护栏和路灯样式统一；
- 河流下游紧邻一座黑色钢桁架桥；
- 对岸是低层历史街区，远处能看到钟楼；
- 城市位于起伏地形中，不是大型都会区。

裁剪对岸建筑和相邻桁架桥做反向图片搜索，可以把候选收敛到俄亥俄州 Marietta。
地图显示，公路桥跨越 Muskingum River，连接东岸 Putnam Street 与西岸
Putnam Avenue；下游钢桁架和河流汇合方向也与题图一致。

[Putnam Bridge 历史标记资料](https://www.hmdb.org/m.asp?m=209248)进一步给出标记
位于 Marietta 的 West Putnam Street 一带，并确认桥名为 Putnam Bridge。
赛事解题记录中的建筑裁剪反搜也得到同一地点。按题目要求的格式填写：

```text
UMDCTF{Putnam Bridge, Marietta, OH 45750}
```

## 方法总结

桥梁定位不应只比较桥本身。相邻铁路桥、河流走向、对岸钟楼和道路连接关系共同组成
更强的地理指纹。反向搜索负责生成候选，地图上的上下游关系与城市轮廓负责排除同名或
外观相近的桥梁，最后再由公开地标资料确认正式名称和邮编。
