# Akira之瞳-1

## 题目简述

附件 `important_work.raw` 是 Windows 7 SP1 x64 内存镜像。需要从进程命令行定位 `work.zip`，解决大文件用 `filescan + dumpfiles` 导出后全为零的问题，转而从相关进程内存中雕刻 ZIP；再从 SAM/SECURITY/SYSTEM 信息恢复用户 NT hash、破解压缩包密码，最后对两张外观相同的图片执行盲水印解码。决定性证据均来自内存镜像，因此从原 PDF 的 `Misc` 调整到 `forensics`。

## 解题过程

先用 Volatility 2 识别镜像：

```bash
volatility -f important_work.raw imageinfo
```

推荐 profile 包含 `Win7SP1x64`，后续固定使用它。枚举进程与命令行：

```bash
volatility -f important_work.raw --profile=Win7SP1x64 pslist
volatility -f important_work.raw --profile=Win7SP1x64 cmdline
```

关键结果是 `important_work.exe`，PID 为 1092；命令行同时出现：

```text
C:\Users\Genga03\Desktop\important_work.exe
C:\Users\Genga03\Desktop\work.zip
```

常规做法是 `filescan` 找到 `work.zip` 再用 `dumpfiles` 导出，但该文件较大，题目镜像中的文件对象虽然保留了正确大小，导出内容却几乎全为 `0x00`。ZIP 数据已被进程使用过，可以改为 dump 整个进程地址空间并做文件雕刻：

```bash
volatility -f important_work.raw --profile=Win7SP1x64 \
  memdump -p 1092 -D ./
foremost 1092.dmp
```

`foremost` 根据 `PK\x03\x04` 等文件签名从 `1092.dmp` 中恢复出 ZIP。压缩包仍有密码，题目提示要求寻找 NTLM/NT hash。运行：

```bash
volatility -f important_work.raw --profile=Win7SP1x64 hashdump
```

其中 Administrator、Guest 使用空密码 NT hash `31d6cfe0d16ae931b73c59d7e0c089c0`，真正相关的登录用户是命令行路径中的 `Genga03`：

```text
Genga03:1001:aad3b435b51404eeaad3b435b51404ee:84b0d9c9f830238933e7131d60ac6436:::
```

对 NT hash `84b0d9c9f830238933e7131d60ac6436` 做字典或在线库查询，得到 ZIP 密码：

```text
20504cdfddaad0b590ca53c4861edd4f5f5cf9c348c38295bd2dbf0e91bca4c3
```

解压后得到 `src.png` 与 `Blind.png`。两张图肉眼几乎一致，文件名 `Blind` 指向盲水印。使用 [BlindWaterMark](https://github.com/chishaxie/BlindWaterMark) 仓库中的旧版 `bwm3.py`，以原图和含水印图共同解码：

```bash
python3 bwm3.py decode src.png Blind.png out.png
```

`out.png` 中显示：

```text
hgame{7he_f1ame_brin9s_me_end1ess_9rief}
```

最终水印图只承载可直接转写的 flag 文本，没有额外视觉分析价值，因此不再作为 WP 图片保留。

## 方法总结

内存取证不能把 `dumpfiles` 当作唯一导出方式：文件对象缓存不完整时，大文件可能只得到零数据，而实际字节仍散落在使用它的进程地址空间，可用 `memdump + foremost` 恢复。随后应把用户名、NT hash、压缩包和图片文件名串成一条证据链；空密码 hash 与目标用户 hash 要明确区分。盲水印阶段需要原图与含水印图成对输入，不能用普通隐写提取工具替代。
