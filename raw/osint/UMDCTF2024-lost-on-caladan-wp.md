# lost on caladan

## 题目简述

题目声称要寻找 Caladan 上“最好的医生”，附件是一处医疗建筑入口的 Google Street View 全景。目标是定位该机构，再从公开执业信息中找出题目刻意安排的医生姓名。

![Baptist Eye Center 入口、接驳车和停车场布局的全景](UMDCTF2024-lost-on-caladan-wp/baptist-eye-center-entrance.jpg)

## 解题过程

先从全景提取可复查的地域线索。画面包含美式 `STOP` 标志、蓝白色院区导向牌、无障碍接驳车和大型停车场，可判断为美国的大型医疗园区。入口附近报刊箱露出的 `amp` 字样可对应 Arkansas Money & Politics，这将范围从全美缩小到阿肯色州。

对比 Little Rock 几家大型医疗机构的卫星图和街景后，Baptist Health Medical Center 周边的停车场布局、蓝白导向牌、米色建筑以及门前接驳车均与附件吻合。具体入口属于 Baptist Eye Center，因此下一步应查找该眼科中心的医生名单，而不是泛搜整个医院。[MapleBacon 的赛时定位记录](https://maplebacon.org/2024/05/umdctf2024-lost-on-caladan/) 保留了卫星图、入口街景和医生目录的逐级核对过程；这些关键判断已经转写在本段中。

在公开医生目录中可以找到姓名非常不寻常的 `Sean Adonis Atreides`。Dune 主题和 `Atreides` 姓氏说明这不是巧合；继续查询其[公开医生资料](https://www.doximity.com/pub/sean-paul-atreides-md)，可同时核对 Little Rock 眼科身份、Baptist Health 地址和完整中间名 `Paul`。完整姓名为：

```text
Sean Paul Adonis Atreides
```

按下划线连接后得到：

```text
UMDCTF{Sean_Paul_Adonis_Atreides}
```

## 方法总结

这题需要连续完成“国家与州 → 医疗园区 → 具体科室 → 医生全名”四级收敛。街景中的小型文字线索适合确定行政区域，建筑和停车场关系用于确认实体，最终姓名则必须回到公开医生目录和执业登记交叉验证；仅提交目录中的省略姓名会缺少中间名。
