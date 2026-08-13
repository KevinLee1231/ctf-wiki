# GreyCTF2022 - flappy-o2

## 题目简述

第二版把状态扩展为 32 位，并让两个 LFSR 相互影响。每个输出字之前还会固定推进 10000 次，再依据第一路状态额外推进第二路，试图让直接动态等待变得缓慢；但状态转移仍完全确定。

## 解题过程

从源码还原两路初始状态，其中主要状态使用种子 `0x1a2b3c4d` 和多项式 `0x80000DD7`。对每个密文字，严格复现以下顺序：固定推进、读取第一路结果、按该结果决定第二路额外步数，最后组合密钥并异或。

```python
for c in encrypted_words:
    for _ in range(10000):
        lfsr1.step()
        lfsr2.step()
    extra = lfsr1.value & extra_mask
    for _ in range(extra):
        lfsr2.step()
    key = combine(lfsr1.value, lfsr2.value)
    output.append(c ^ key)
```

这里无需符号执行，直接重放即可。官方数据解得：

```text
grey{y0u_4r3_v3ry_g00d_4t_7h1s_g4m3_c4n_y0u_t34ch_m3_h0w_t0_b3_g00d_ef4bd282d7a2ab1ebdcc3616dbe7afb}
```

## 方法总结

大量循环只增加运行时间，不增加状态熵。只要种子、更新函数和消费顺序公开，输出仍可预测。实现时应以源码的一次完整循环为单位记录状态变化，避免把“先取值再推进”和“先推进再取值”混淆。
