# Not again! I've been BitLockered out of my own computer!

## 题目简述

附件是 Windows 内存镜像，目标是恢复三条仍驻留内存的 BitLocker Full Volume Encryption Key（FVEK）。题目允许三条密钥以任意顺序提交。

## 解题过程

先用 Volatility 2 识别镜像配置：

```text
python2 vol.py -f 20240327.mem imageinfo
python2 vol.py -f 20240327.mem kdbgscan
```

`kdbgscan` 给出的匹配配置为 `Win10x64_19041`。将 [Volatility BitLocker 插件](https://github.com/elceef/bitlocker) 放入插件目录后执行：

```text
python2 vol.py -f 20240327.mem --profile=Win10x64_19041 bitlocker
```

输出中可能还有旧版结构的误报；本题要取标注为 Windows 8 及以上布局的三条有效 FVEK：

```text
AES-256  91c75f658705c36090f03779cacb056179e16316ee4af1e90d0f84e090b88d8b
AES-128  91d4cceb5bf238cb3cb96367314773f5
AES-128  968052b6b247b32f6cfecce39749785f
```

任选顺序拼接，例如：

```text
byuctf{968052b6b247b32f6cfecce39749785f_91c75f658705c36090f03779cacb056179e16316ee4af1e90d0f84e090b88d8b_91d4cceb5bf238cb3cb96367314773f5}
```

## 方法总结

挂载中的全盘加密卷必须在内存里保留可用密钥，因此内存采集可能绕开口令恢复。实操时先可靠识别 Windows profile，再根据插件输出的算法和结构版本过滤候选，不能把扫描到的每段相似字节都当作有效 FVEK。
