# closing-bell

## 题目简述

远端输出一份约 16 MB 的“市场行情”文本，其中隐藏了本次连接专属的 ELF。四个行情字段由同一字节流派生，还混有丢包、噪声和诱饵行。恢复 ELF 后，还要从程序配置中求出动态 settlement key，再提交该 key 获取 shell。

仓库提供的参考脚本完成“行情文本到 ELF”的部分；ELF 中的 key 恢复可由 `artifact.c` 和 `service.py` 直接推导。

## 解题过程

### 从行情恢复字节流

真实行可由 `LOT` 映射回连续索引。每个 venue 都使用模 257 的仿射通道：

$$
y_v=a_vx+b_v\pmod{257},
$$

再把 $y_v$ 混入 `YES`、`DEPTH`、`SPREAD`、`IMB` 四个字段。前 256 组行情对应公开的完整字节梯：

$$
x_i=(73i+41)\bmod256.
$$

利用这些校准样本，可以为每个 venue 穷举 $a_v,b_v$，并用四字段误差评分排除 `LOT` 碰撞产生的诱饵。对后续每个索引，先从 `YES` 反推出候选 $y_v$，再求

$$
x=a_v^{-1}(y_v-b_v)\pmod{257}.
$$

仅保留 $x<256$ 且四字段总误差足够小的候选，最后按多个 venue 的多数票恢复该字节。

### 解密并确认 ELF

恢复的数据以明文 `CBELL24/ELF2\n` 开头，后面是 24 位种子生成的异或流。枚举 $0\leq s<2^{24}$，只解密短头部并检查：

```text
CBELL-ELF-V2\0
```

命中后读取长度和 FNV 风格校验值，完整解密并同时验证 ELF 魔数与 checksum，即可排除假阳性。

### 从 ELF 恢复本次 key

在 ELF 中搜索 16 字节配置标记 `CBELLPATCHCFG01!`。其后依次是 8 字节目标哈希和 64 字节 `sealed_book`。服务端生成 key 时先把 `KEY{...}` 零填充到 64 字节，再把 `roll_book` 逆运算 19488 轮写入 ELF。

因此对提取出的 `sealed_book` 正向执行 `tick=0..19487` 的 `roll_book`，结果就是原始 key 加零填充。去掉末尾 NUL 后提交本次动态 key，进入 shell并读取：

```text
UMDCTF{the_closing_bell_is_soon-get-y0ur-b3ts-IN!!!}
```

## 方法总结

本题分为三层：用校准字节拟合多 venue 仿射信道、穷举 24 位流密码种子恢复 ELF、再逆向 ELF 内可逆的 book 状态更新。每层都有独立校验：多字段评分、ELF 魔数与 checksum、目标哈希。由于最主要的工作是从伪装成行情的隐蔽通道重组载荷，最终归入 Stego；ELF 逆向是后续阶段。利用分层校验逐步收紧，比直接在 16 MB 文本中搜索可打印字符串可靠得多。
