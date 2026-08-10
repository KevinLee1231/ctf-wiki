# Hijacked

## 题目简述

附件是一份约 2 MB 的 PCAP。题面提到失控遥控车和 West Vancouver，流量中还访问了 GPX 1.1 规范页面。两段明文 HTTP 响应包含按顺序记录的纬度、经度和第三列数值；把前两列恢复为 GPX 轨迹后，路径在地图上画出 flag。

## 解题过程

先查看协议分层。抓包共 579 帧，除 DNS、TLS 外只有少量 HTTP，其中包括对 topografix.com/GPX/1/1/ 的请求，以及 ix.io/3MDT、ix.io/3MDU 两个 text/plain 响应。后两者的每行形如：

~~~text
49.26133 -123.11518 38.66
49.26050 -123.11520 44.85
49.26050 -123.11508 45.01
~~~

可以在 Wireshark 中依次 Follow HTTP Stream 或 Export Objects，也可以让 tshark 输出 http.file_data 后做十六进制解码。必须按数据在抓包中的先后顺序拼接两个响应；第三列不参与坐标绘制。

把每行前两列写为 GPX track point：

~~~python
from xml.sax.saxutils import escape

points = []
for line in coordinate_text.splitlines():
    lat, lon, _ = line.split()
    points.append((lat, lon))

with open("flag.gpx", "w", encoding="utf-8") as out:
    out.write('<?xml version="1.0" encoding="UTF-8"?>\n')
    out.write('<gpx version="1.1" creator="solver" ')
    out.write('xmlns="http://www.topografix.com/GPX/1/1/">\n<trk><trkseg>\n')
    for lat, lon in points:
        out.write(f'<trkpt lat="{escape(lat)}" lon="{escape(lon)}"/>\n')
    out.write("</trkseg></trk></gpx>\n")
~~~

在支持 GPX 的地图查看器中打开后，连续轨迹组成大小写混合的字符。题面特别提醒检查大小写；官方接受多种易混淆字符，清晰版本为：

~~~text
maple{mI5c_bAc0n_Fr0M_0utER_5pAc3}
~~~

## 方法总结

PCAP 题应先做协议和内容类型盘点，再决定是否深挖加密流量。本题真正证据位于少量明文 HTTP 对象中，GPX 规范请求则给出解释坐标的格式提示。恢复轨迹时顺序、经纬度方向和大小写都不能丢；外部网站只负责显示，关键数据和构造方法应保留在本地解题链中。
