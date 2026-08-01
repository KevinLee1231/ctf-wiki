# BYUCTF 2022 - Probably

## 题目简述

远程服务每次以 FIGlet 大字输出一个“可能正确”的 flag。源码 `probably.py` 对每个字符独立处理：以 25% 概率保留真实字符，否则从小写字母、数字和下划线中随机选择一个字符。

## 解题过程

重复连接服务并把每次 FIGlet 输出还原为等长字符串。人工少量采样时可直接辨认 FIGlet 字形；自动化时则以服务使用的同一 FIGlet 字体预渲染字符模板，再按固定字符宽度匹配。对固定位置统计频数即可：真实字符单次出现概率约为

$$
0.25 + 0.75/37 \approx 0.2703,
$$

而任一错误字符只有 `$0.75/37 \approx 0.0203$`。因此样本数增加后，真实字符会明显成为众数。

流程可以概括为：

```python
from collections import Counter

samples = [...]  # 多次读取并从 FIGlet 还原出的字符串
flag = ''.join(Counter(col).most_common(1)[0][0]
               for col in zip(*samples))
print(flag)
```

花括号不在随机字符集合中，但源码仍会以 25% 概率原样输出；多采几次即可确认边界。按列取众数得到：

```text
byuctf{what_are_the_chances_3eep3fcs}
```

## 方法总结

这是独立重复采样下的统计恢复。错误字符分散在 37 个候选上，真实字符虽每次未必出现，却在频率上有数量级优势；按位置聚合比试图等待一次完整正确输出有效得多。
