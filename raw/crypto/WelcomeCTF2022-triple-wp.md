# triple

## 题目简述

附件包含密文 `flag.pptx` 和三个与密文等长的文件 `1`、`2`、`3`。生成脚本把原始 PowerPoint 的每个字节依次与这三个文件异或，题目利用的是 XOR 的可逆性，而不是 PowerPoint 文件格式本身的漏洞。

## 解题过程

设原文件为 $P$，三个掩码为 $K_1,K_2,K_3$，则密文为：

$$
C=P\oplus K_1\oplus K_2\oplus K_3.
$$

由于 $x\oplus x=0$ 且 XOR 满足交换律和结合律，再异或同样的三个掩码即可得到 $P$：

```python
from pathlib import Path

c = Path("flag.pptx").read_bytes()
k1 = Path("1").read_bytes()
k2 = Path("2").read_bytes()
k3 = Path("3").read_bytes()

plain = bytes(a ^ b ^ d ^ e for a, b, d, e in zip(c, k1, k2, k3))
Path("plaintext.pptx").write_bytes(plain)
```

恢复出的文件以 ZIP/PPTX 的 `PK` 文件头开头，可以正常打开。对照原始幻灯片检查后，唯一一页中的文本为：

```text
greyhats{7hr33_P1eC3_x0RIg1N4l_m34L_p1e4S3}
```

幻灯片背景只起装饰作用，Flag 是可复制文本，因此无需把页面截图作为 WP 图片保留。

## 方法总结

重复执行 XOR 不会自动增加安全性；只要所有掩码都随密文一同给出，就能按任意顺序消去。处理二进制文件时应逐字节运算，并在恢复后同时验证文件头、容器结构和实际打开效果，避免只凭扩展名判断成功。
