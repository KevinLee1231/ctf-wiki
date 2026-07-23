# Carrier Crisis

## 题目简述

题目给出一串浮点采样数据，并提示 751 MHz 载波。仓库中的 GNU Radio 流图显示，发送端把循环读取的 flag 字节转换为浮点数，再乘以余弦载波；接收端只需用相同载波完成相干解调并转回字符。

## 解题过程

物理载波频率虽然写成 751 MHz，但文件中保存的是采样后的浮点值，流图的实际采样率为 32 kHz。发送模型可以写成：

$$
s[n]=m[n]\cos(2\pi f_cn/f_s)
$$

仓库给出的解密流图没有再做一次乘法，而是用相同相位和频率的余弦逐样本相除，从而直接撤销发送端乘法，再执行 float-to-char：

```text
File Source (float)
  -> Divide by Local Oscillator
  -> Float To Char
  -> File Sink
```

仓库已经给出对应的 `decrypt.grc`，打开后把输入指向 `prod/message` 并运行，即可在输出中看到循环出现的：

```text
UMDCTF-{wh@ts_@_c@rr13r}
```

## 方法总结

无线题中的高射频提示不代表必须拥有真实射频前端。只要附件是离线 IQ 或浮点采样，就应优先还原 GNU Radio 信号链；本题的关键是用完全相同的离散本振撤销逐样本乘法，而不是直接把浮点值强制解释为字符。
