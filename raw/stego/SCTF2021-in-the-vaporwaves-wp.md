# in_the_vaporwaves

## 题目简述

附件 `c.wav` 是 44.1 kHz、16 位、双声道音频。直接播放只有正常音乐，但左右声道的主要波形互为反相；生成脚本把同一段隐藏信号分别叠加到两个声道，使音乐在求和时相消，而隐藏信号被保留下来。解题目标是进行无损声道合成，再把恢复出的 500 Hz 音调按时长解为 Morse 码。

## 解题过程

### 合并左右声道

设原音乐声道近似满足 $R_c(t)=-L_c(t)$，隐藏信号为 $h(t)$，题目构造的两个声道可以写成

$$
L(t)=L_c(t)+h_L(t),\qquad R(t)=R_c(t)+h_R(t).
$$

因此

$$
L(t)+R(t)\approx h_L(t)+h_R(t).
$$

逐帧读取两个有符号 16 位采样并相加即可。官方简稿把输出采样宽度写成 1 字节，却仍使用 `struct.pack("h", ...)` 写入 2 字节；复现时应把采样宽度保持为 2，并对和做 16 位截断保护：

```python
import struct
import wave

with wave.open("c.wav", "rb") as source:
    assert source.getnchannels() == 2
    assert source.getsampwidth() == 2

    with wave.open("recovered-morse.wav", "wb") as target:
        target.setnchannels(1)
        target.setsampwidth(2)
        target.setframerate(source.getframerate())

        for _ in range(source.getnframes()):
            left, right = struct.unpack("<hh", source.readframes(1))
            value = max(-32768, min(32767, left + right))
            target.writeframesraw(struct.pack("<h", value))
```

处理后的单声道文件约 40.4 秒，原音乐大幅衰减，剩下一组清晰的等频音调。无需保留频谱截图：载频位置和时长都可以直接量化。

### 按时长还原 Morse

对 500 Hz 附近做窄带能量检测并二值化后，连续有声段主要分为约 0.1 秒和 0.3 秒，分别对应点与划；符号内间隔约 0.1 秒，字符间隔约 0.4 秒。得到：

```text
... -.-. - ..-. -.. . ... .---- .-. ...-- ..--.-
-.. .-. .. ...- . ... ..--.- .. -. - ----- ..--.-
...- .- .--. --- .-. .-- .--.-. ...- . ...
```

按国际 Morse 表解码为：

```text
SCTFDES1R3_DRIVES_INT0_VAPORW@VES
```

其中题目音频省略了花括号的 Morse 表达，按比赛 flag 格式补回后为：

```text
SCTF{DES1R3_DRIVES_INT0_VAPORW@VES}
```

## 方法总结

这类音频题应先比较声道间的相位与相关性，而不是一开始就只看全局频谱。左右声道求和能消除反相的封面音频并强化同相隐藏信号；恢复载波后，再用有声段长度和空白间隔区分 Morse 的点、划、字符边界。转写时保留采样率、采样宽度、求和公式和定时阈值，已经足以复现，因此原 WP 中的波形与 Morse 对照截图没有继续归档。
