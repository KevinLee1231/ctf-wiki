# Corrupted Cores

## 题目简述

题目仍使用 `GLaDOS_Network.pcapng`。一组 ICMP 包的源 IPv4 地址明显不符合正常通信规律；每个地址的四个八位组被用作四个数据字节，最终结果还经过一层 Base64 编码。

## 解题过程

过滤对应 ICMP 流量，记录序号和源地址。按 ICMP 序号排序后，把每个源地址 `a.b.c.d` 转为四个字节 `[a, b, c, d]` 并连接。这样得到的是一串 Base64 文本，而不是最终 flag。

可复现的核心逻辑如下：

    ordered = sorted(packets, key=lambda p: p.icmp_sequence)
    encoded = bytes(
        octet
        for packet in ordered
        for octet in map(int, packet.source_ip.split("."))
    )
    flag = base64.b64decode(encoded)

解码结果为：

    byuctf{Th3_P4rt_Wh3r3_H3_K!lls_Y0u}

这里必须保留地址中的前导零语义为数值字节，并以协议序号而非显示顺序重组。

## 方法总结

源地址也可能是隐蔽信道。判断依据是地址变化模式与正常网络拓扑不符，且四个八位组能稳定映射为可打印编码。恢复后若仍呈现另一种明确编码，应继续解码并用格式验证，而不是把中间字符串当作答案。
