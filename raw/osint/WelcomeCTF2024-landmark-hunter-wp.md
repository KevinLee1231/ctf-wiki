# Landmark Hunter

## 题目简述

附件包含大量 Base64 字符串。每行解码后都是 Google Encoded Polyline，绝大多数坐标落在海洋等无关位置，只有一个异常点对应目标地标。目标是批量还原坐标，再通过公开地图反向地理编码确认名称。

## 解题过程

逐行先做 Base64 解码，再按 Encoded Polyline 的变长整数、ZigZag 差分规则恢复纬度和经度。官方 solver 指出异常位于第 236 行；该行：

```text
d|ioA`~wyL
```

解码后的首个坐标为：

```text
-13.16307, -72.54513
```

将坐标提交给 OpenStreetMap Nominatim、Google Maps 或其他公开地图服务进行反向地理编码，可定位到秘鲁的 Machu Picchu。自动处理时应尊重 Nominatim 的速率限制，例如每次请求至少间隔一秒，并保存坐标与返回名称便于复核。

答案按小写和下划线格式提交：

```text
grey{machu_picchu}
```

## 方法总结

该题包含两层表示：Base64 只负责把文本还原为 polyline，polyline 再恢复坐标。OSINT 结论不能停在坐标本身，还需用公开地图交叉确认地标名称及标准拼写。
