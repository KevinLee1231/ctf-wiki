# notsogeo

## 题目简述

题目展示一幅可拖动的 Google Street View 全景图，要求在地图上提交精确位置。服务端按提交坐标与目标坐标的距离判断，距离格式化为 `0.0` 时返回 flag。

页面没有直接公开经纬度，却把 Street View 的 panorama ID 放进了静态 JSON 和瓦片请求中；该 ID 可以反查全景元数据。

## 解题过程

查看页面加载的 `/info.json`：

```json
{
  "chall": {
    "panoId": "E79vkEu2pHDfiUkvUWHciA",
    "width": 32,
    "height": 16
  }
}
```

前端随后把同一个值放进 Street View 瓦片 URL 的 `panoid` 参数。panorama ID 唯一标识一个街景拍摄点，可通过 Google Street View metadata 接口或支持该 ID 的地图客户端查看其 `location.lat` 与 `location.lng`。查询得到：

```text
61.590371, 41.4454648
```

前端 `submit()` 以 JSON 数组发送坐标。直接复现请求：

```http
POST /chall/submit HTTP/1.1
Content-Type: application/json

[61.590371, 41.4454648]
```

服务端首先检查两组浮点数是否完全相等；相等时距离直接置为 0，所以无需研究后面的球面距离公式。响应为：

```text
yes, UMDCTF{n0t_th3_b35t_1dea_exp0s!ng_th3_p4nor@ma_ID_3897334}
```

## 方法总结

- panorama ID 不是普通的无意义前端标识，它能映射到街景元数据和拍摄坐标。
- 地图、瓦片和媒体类应用应检查静态 JSON、网络参数及客户端状态，敏感元数据不能仅靠界面隐藏。
- 本题应提交原始精度的经纬度；手工点击地图会引入误差。
- 服务端存在直接相等的快速分支，只要坐标完全一致就稳定返回 `0.0`。
