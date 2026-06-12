# Robust

## 题目简述

本题是多层流量取证与音频隐写题。附件包含 QUIC/HTTP3 流量和 Firefox SSL Key Log；导入 key log 后可解密 HTTP3，提取加密 HLS 直播流、解出 TS 视频，再从频谱中的高频声信号恢复数据。

题目附件包含 QUIC/HTTP3 流量、Firefox SSL Key Log 和后续多层隐写载体。完整证据链是：QUIC + TLS 1.3 HTTP3 解密 -> HLS m3u8/key/ts 提取 -> AES-128-CBC 解密视频 -> quiet ultrasonic 解码 -> Base64/ZIP -> 歌词明文攻击 zip 密码 -> 空白字符隐写 -> Base85 解码 flag。

## 解题过程

题目生成脚本/附件说明见 [`zhouweitong3/d3ctf_Robust`](https://github.com/zhouweitong3/d3ctf_Robust)。下文保留了该仓库对应的载体链路：HTTP/3 流量、HLS 解密、quiet ultrasonic、歌词明文攻击和 Unicode 空白字符隐写。

打开 `cap.pcapng` 后可以看到流量基本都是 QUIC。附件同时给了 `firefox.log`，也就是 Firefox 访问时生成的 SSL Key Log；结合 QUIC 强制 TLS 1.3 的特征，可以判断这是 HTTP/3 流量。把 key log 导入 Wireshark 后，HTTP/3 数据包即可被解密并过滤查看：

![Wireshark 解密后过滤出的 HTTP3 数据包](<./D3CTF2021-robust-wp/wireshark_http3_packets.png>)

前几百个包主要是在加载网页和 `hls.js`。继续向后看，在一个 HTTP/3 响应里能找到 m3u8 playlist，并且该 HLS 直播流使用 AES-128 加密；随后在后续响应中找到解密 key，保存为 `enc.key`。

切片数量较多，不适合手动复制。可以用 `pyshark` 把 HTTP/3 data frame payload 按顺序提取出来，先合并成一个 TS 文件，再按 HLS 的 AES-128-CBC 规则解密：

```python
import os
import pyshark

cap = pyshark.FileCapture(
    "cap.pcapng",
    override_prefs={"ssl.keylog_file": os.path.abspath("firefox.log")},
)

with open("output.ts", "wb") as fd:
    for i in range(678, 18706):
        try:
            if int(cap[i].http3.frame_type) == 0:
                fd.write(cap[i].http3.frame_payload.binary_value)
        except Exception:
            continue
```

构造本地 `index.m3u8`，把解密密钥 URI 改成相对路径：

```text
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-KEY:METHOD=AES-128,URI="enc.key",IV=0x00000000000000000000000000000000
#EXTINF:10000
output.ts
#EXT-X-ENDLIST
```

随后用 FFmpeg 解密：

```bash
ffmpeg -allowed_extensions ALL -i index.m3u8 -c copy outdec.ts
```

也可以用 OpenSSL 手动解密。需要注意 HLS AES-128 遇到过长 key 时只截取前 128 bit。`xxd -P enc.key` 后取开头 16 字节对应的 hex key：

```bash
openssl aes-128-cbc -d -in output.ts -out out.ts \
  -iv 00000000000000000000000000000000 \
  -K 34363238656561363031396632323631 \
  -nosalt
```

解密出的 TS 可以直接播放。用 Adobe Audition 看频谱后，可以看到明显的高频信息：

![Adobe Audition 中的初始频谱](<./D3CTF2021-robust-wp/audition_initial_spectrogram.png>)

把数据调制到音频频段，很容易联想到 modem 类方案。原题提示指向 [`quiet`](https://github.com/quiet/quiet)：发送端把字节流调制成音频信号，接收端按相同 profile 解调回字节流。频谱中稳定集中在 19KHz 附近的信号不是噪声，而是 quiet ultrasonic profile 生成的隐写层。编译 `quiet/libfec` 和 `quiet/quiet` 后，回到 Audition 对照频谱中心频率：

![以 19KHz 为中心的 ultrasonic 频谱](<./D3CTF2021-robust-wp/quiet_ultrasonic_spectrogram.png>)

将待解码音频转成 WAV，并保持原采样率和较高量化位数以减少损失，然后使用 quiet 示例程序解码：

```bash
./quiet_decode_file ultrasonic out.txt
```

输出是 Base64。解码后出现 `PK` 文件头，保存为 ZIP，但该 ZIP 有密码。文件名线索指向网易云音乐歌词；歌曲 ID 是 `1818031620`，对应页面为 <https://music.163.com/song?id=1818031620>。可以从网易云客户端离线缓存中取歌词：

```text
%LOCALAPPDATA%\Netease\CloudMusic\webdata\lyric
```

也可以通过歌词接口获取同一份 JSON：

```text
http://music.163.com/api/song/lyric/?id=1818031620&lv=1&kv=1&tv=1
```

把歌词保存为 `lyric-1818031620.json`，使用 Advanced Archive Password Recovery 或 `pkcrack` 做 ZIP 已知明文攻击。成功解密后得到 txt 文档，换其他软件或十六进制编辑器查看可发现空白/零宽字符隐写。

最后用 Unicode 空白字符隐写工具 <https://330k.github.io/misc_tools/unicode_steganography.html> 解码。解码前先确认使用的是哪几类空白/零宽字符，再选择匹配设置；本题得到的是 Base85：

```text
<~A2@_;ApZ7(GA0]MC.i&:F%'t#:JXSd=tj-$>'EtK0m.1t0i38~>
```

Base85 解码得到 flag：

```text
d3ctf{1IwiKUjKcUsEn0OOJZZ0ZsZwUX1uiC1P}
```

## 方法总结

- 核心技巧：把网络取证、媒体容器、音频调制解码和文本隐写串成证据链，每一层只提取下一层必要 artifact。
- 识别信号：QUIC 流量旁边给出 Firefox SSL Key Log、HTTP3 中出现 m3u8/HLS、视频频谱在 19KHz 附近有规则高频信号、ZIP 密码线索指向歌曲歌词。
- 复用要点：HLS AES-128 key 过长时只取前 128 bit；quiet ultrasonic 解码前要保持采样率和量化精度；歌词明文攻击 zip 时要注意压缩算法参数。

