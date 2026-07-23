# UMDCTF 2025 - gerard-manley-hopkins

## 题目简述

题目给出一张俄亥俄州居民区的 360 度街景，要求提交拍摄位置所在道路的完整地址。

![Wellsville Hillcrest Road 的 360 度街景：双黄线路面、坡地住宅和架空线用于核对拍摄道路](UMDCTF2025-gerard-manley-hopkins-wp/hillcrest-road.jpg)

## 解题过程

图片没有 GPS 等可直接使用的元数据。住宅样式、双黄线和架空线只能说明这是美国小镇，
不足以唯一定位；真正可搜索的线索是路边垃圾桶上的 `Dailey's` 标志和电话号码
`532-4667`。

搜索公司名与号码，可以把服务商定位到俄亥俄州 Wellsville。随后在地图和街景中查找
相同的陡坡、弯道、架空线、房屋布局，并用附近可见的门牌 `1121` 与 `1116` 缩小
范围。这两个门牌对应相邻的 Mick Road 一带，但全景相机实际位于与其连接的
Hillcrest Road 上，不能把用于定位的邻街误当成答案。

[赛事解题记录](https://davidpirneci.com/posts/umdctf-2025-writeup/)保留了这条
可复核的证据链：垃圾桶上的 Dailey's 电话先确定 Wellsville，门牌和街景再确定
Mick Road 邻接点，最后读取相机所在道路。邮政地址为：

```text
Hillcrest Rd, Wellsville, OH 43968
```

因此 flag 是：

```text
UMDCTF{Hillcrest Rd, Wellsville, OH 43968}
```

## 方法总结

普通居民区缺少独特地标时，应优先寻找带地域性的服务商、电话号码和门牌。电话号码
用于确定城市，门牌用于锁定街段，最终仍要回到地图确认全景相机究竟落在哪条道路上；
“画面中看到的邻街地址”和“题目要求的拍摄道路”并不一定相同。
