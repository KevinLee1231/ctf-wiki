# UMDCTF 2025 - beauty

## 题目简述

题目给出一张城市全景图，要求找出该全景的拍摄者姓名，而不是只定位城市。

![Scioto 河、前景绿地与 Columbus 市中心天际线的 360 度全景，用于确认城市和拍摄方向](UMDCTF2025-beauty-wp/columbus-skyline.jpg)

## 解题过程

图片没有可用的 GPS、作者或描述元数据，必须从画面本身开始。画面中的宽阔河流、
河岸公园和市中心天际线是第一组线索；右侧高楼群中的 LeVeque Tower，以及图像
底部的 `COSI` 标记，可以把地点收敛到俄亥俄州 Columbus 的 Scioto 河岸。

[Columbus 官方旅游页面](https://www.experiencecolumbus.com/blog/post/best-skyline-views-in-columbus/)
说明 COSI 后方的 Genoa Park 正是当地典型的城市天际线观察点。确定城市和朝向后，
在地图中沿 COSI 一侧的河岸检查 360 度全景，而不是继续对整张图做宽泛反向搜索。

找到画面中河道、桥梁、法院建筑和高楼相对位置完全一致的用户全景后，查看全景信息
面板中的贡献者署名，显示为 `Neil Larimore`。赛事期间的
[另一份解题记录](https://krypton.ninja/ctfs/umdctf-2025-ctf-write-up/)
也保存了相同的定位链：先由天际线确认 Columbus，再在河对岸唯一匹配的全景中读取
作者名。由此得到：

```text
UMDCTF{Neil Larimore}
```

## 方法总结

本题最终答案属于平台对象的作者属性，地理定位只是中间步骤。有效流程是先用稳定地标
确定城市和观察方向，再在小范围内逐个核对用户全景并读取署名。反向图片结果只能用于
生成候选，河岸形状、桥梁和建筑相对位置的视觉一致性才是确认依据。
