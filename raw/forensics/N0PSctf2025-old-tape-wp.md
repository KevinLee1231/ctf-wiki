# Old Tape

## 题目简述

附件 ZIP 中是一段能够播放、但画面持续出现拖影和块状故障的 AVI 视频。需要识别 datamoshing 痕迹，修复被重复插入的视频帧，再从恢复后的画面读取 flag。

## 解题过程

视频的容器结构仍然有效，故障主要发生在帧序列。画面保留旧图像轮廓并被后续运动不断拉伸，这是重复 P 帧造成的典型 datamoshing 效果：P 帧主要保存相对参考帧的运动变化，重复应用相同数据会让预测误差持续累积。

AVI 中压缩视频数据常使用 FourCC `00dc` 标识。其原始字节为：

```text
30 30 64 63
```

将文件按该标识切分后，可以发现多个帧片段在字节级完全相同。保留每个片段第一次出现的位置、删除后续重复项，再按原分隔符连接即可得到修复版视频：

```python
from pathlib import Path

FRAME_MARKER = b"00dc"
data = Path("challenge.avi").read_bytes()
parts = data.split(FRAME_MARKER)

seen = set()
with open("fixed.avi", "wb") as output:
    output.write(parts[0] + FRAME_MARKER)

    for frame in parts[1:]:
        if frame in seen:
            continue
        seen.add(frame)
        output.write(frame + FRAME_MARKER)
```

修复后播放 `fixed.avi`，原本被拖影遮挡的文字可以正常辨认：

```text
N0PS{4v1_f0rM4t_h4Z_n0_5eCr37_4_U_4nYM0r3}
```

最终画面只承担文字展示作用，没有额外的空间或视觉分析信息，因此直接把内容转写为文本，不再保留视频截图。

## 方法总结

本题应先区分“容器损坏”和“编码帧被恶意重排或重复”。文件能够正常播放且故障呈运动累积特征，更符合 datamoshing。对题目给定样本，按 `00dc` 分隔并做稳定去重即可恢复；所谓稳定去重，是只删除后续重复项而不改变首次出现顺序，否则参考帧关系仍会被破坏。
