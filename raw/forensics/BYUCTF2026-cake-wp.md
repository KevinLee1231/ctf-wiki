# Cake

## 题目简述

题目给出 `GLaDOS_Network.pcapng`，要求从网络流量中恢复隐藏信息。线索位于普通 HTTP 请求的 Cookie，而不是加密载荷或文件雕刻。

## 解题过程

在 Wireshark 中使用 `http` 过滤器检查请求头，可以发现一个名为 `cake` 的 Cookie，其值是一段符合 Base64 字符集的文本。也可以用 tshark 提取所有 Cookie 字段后再筛选：

    tshark -r GLaDOS_Network.pcapng -Y http.cookie -T fields -e http.cookie

取出 `cake=` 后的值并做一次 Base64 解码，结果直接是：

    byuctf{Th3_C4k3_!s_4_L!3_HTC56zeE}

截图中的 Wireshark 表格只展示了同样的文本字段，没有额外空间或视觉信息，因此题解直接记录过滤条件和解码步骤即可。

## 方法总结

分析明文 HTTP 流量时应系统检查 URI、查询参数、请求体、Cookie 与自定义头。发现疑似编码文本后，需要结合字符集、长度和解码结果验证，而不能仅凭外观认定为 Base64。
