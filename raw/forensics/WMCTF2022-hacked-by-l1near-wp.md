# Hacked_by_L1near

## 题目简述

题目给出 Tomcat WebSocket 通信期间的流量，题面说明服务器被写入了 WebSocket 内存马，并要求从抓到的通信中恢复 L1near 写入的 flag。流量中 WebSocket 握手协商了 `permessage-deflate`，后续消息的 RSV1 位表明 payload 被按消息压缩；客户端到服务端的帧还带有 WebSocket mask。解题目标是还原帧结构、去 mask、按 raw DEFLATE 解压，提取内存马写文件命令和回显中的 flag。

## 解题过程

这里我们可以知道是tomcat的websocket，基本上都是默认开启了permessage-deflate，然后分析数据包我们也可以知道其中的permessage-deflate的开启情况。RFC 7692 的关键点是：`permessage-deflate` 在握手阶段通过 `Sec-WebSocket-Extensions` 协商，压缩单位是完整 WebSocket message，压缩帧通常用 RSV1 标记，payload 使用 DEFLATE 变换；因此脚本要先处理 WebSocket 帧头、长度字段和 mask，再用 `zlib.decompress(data, -15)` 这类 raw deflate 方式解压。中间有些数据会失真，无法全部解出，但可解出的部分已经足够恢复写入 flag 的命令和回显。

exp.py:

~~~python
from Crypto.Util.number import *
import zlib

def unmark(masked_data,mask_key,payload_length):
    res = b''
    for i in range(len(masked_data)):
        res += long_to_bytes(masked_data[i] ^ mask_key[i % 4])
    payload = hex(bytes_to_long(res))[2:]
    fin_payload = _fill(payload,payload_length)
    return fin_payload

def _fill(payload,payload_length):
    if payload.__len__()!=payload_length*2:
        payload = payload.zfill(payload_length*2)
    payload = payload[0] + hex(int(payload[1],16)+1)[2:] + payload[2:]
    return payload

f = open('1.txt','r').read().split('\n')
# print(f)
for ff in f:
    try:
        websocket_info = bin(int(ff[:4],16))[2:]
        mode = websocket_info[1]
        if mode == '1':
            print('permessage-deflate')
        else:
            # print('no permessage-deflate')
            continue
        payload_length = int(websocket_info[-7:],2)
        
        mask = websocket_info[8]
        
        if mask == '1':
            print('marked')
            if payload_length != 0:
                if payload_length == 126:
                    payload_length = int(ff[4:8],16)
                    print('payload_length:',payload_length)
                    mask_key = long_to_bytes(int(ff[8:16],16))
                    # print(ff[8:16])
                    masked_data = long_to_bytes(int(ff[16:],16))
                    payload = unmark(masked_data,mask_key,payload_length)
                    print('payload:',payload)
                    data = long_to_bytes(int(payload,16))
                    fin = zlib.decompress(data,-15)
                    print(fin.decode())
                    
                else:
                    print('payload_length:',payload_length)
                    mask_key = long_to_bytes(int(ff[4:12],16))
                    # print(mask_key)
                    masked_data = long_to_bytes(int(ff[12:],16))
                    payload = unmark(masked_data,mask_key,payload_length)
                    print('payload:',payload)
                    data = long_to_bytes(int(payload,16))
                    fin = zlib.decompress(data,-15)
                    print(fin.decode())
                    
            else:
                print('payload_length:',payload_length)
            print()
        else:
            print('unmarked')
            if payload_length != 0:
                if payload_length == 126:
                    payload_length = int(ff[4:8],16)
                    print('payload_length:',payload_length)
                    payload = hex(int(ff[8:],16))[2:]
                    fin_payload = _fill(payload,payload_length)
                    print('payload:',fin_payload)
                    data = long_to_bytes(int(fin_payload,16))
                    fin = zlib.decompress(data,-15)
                    print(fin.decode())
                    
                else:
                    print('payload_length:',payload_length)
                    payload = hex(int(ff[4:],16))[2:]
                    fin_payload = _fill(payload,payload_length)
                    print('payload:',fin_payload)
                    data = long_to_bytes(int(fin_payload,16))
                    fin = zlib.decompress(data,-12)
                    print(fin.decode())
                    
            else:
                print('payload_length:',payload_length)
            print()
    except:
        print()
        pass
~~~

关键解压结果中可以看到一次写文件/拷贝行为：

```text
echo "WMCTF{...uuid...}/home/test/flag&&uuid>/home/test/flag&&echo "}" >>/home/test/flag
```

后续服务端回显补全了 flag 内容：

```text
WMCTF{
b4bb6968-0b0a-11ed-b5af-5f6e282b4631
}
```

因此最终 flag 为 `WMCTF{b4bb6968-0b0a-11ed-b5af-5f6e282b4631}`。

## 方法总结

本题核心是按协议层级拆 WebSocket 压缩帧：先解析 FIN/RSV/opcode 和 payload length，再按 mask key 还原客户端 payload，最后用 raw DEFLATE 解压 `permessage-deflate` 数据。

识别信号是 Tomcat WebSocket、握手里出现 `permessage-deflate`、数据帧 RSV1 置位、客户端帧带 mask、CyberChef 对部分 payload 做 raw inflate 后能看到 shell 命令片段。

复现时注意 payload length 为 126 时长度字段会扩展，mask key 的偏移随长度字段变化；`zlib.decompress` 的窗口参数也要按 raw deflate 处理。若个别帧损坏或上下文不完整，优先从可解出的写文件命令和回显文本拼出 flag。

参考资料要点：RFC 7692 定义了 WebSocket per-message compression 机制，说明压缩扩展在握手中协商、使用 RSV1 标记压缩消息、`permessage-deflate` 基于 DEFLATE。原文见 https://datatracker.ietf.org/doc/html/rfc7692。
