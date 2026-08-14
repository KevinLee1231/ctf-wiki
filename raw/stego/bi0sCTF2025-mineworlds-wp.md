# Mineworlds

## 题目简述

这是一道以 Minecraft 世界与资源包为载体的隐写题。官方短题解给出的链路是：先穷举本地 SHA-256 截断哈希以定位服务器已加载区域，再根据游戏内提示把范围缩小到数个方块，最终发现 Polished Basalt 的顶面贴图被替换成二维码；二维码给出口令，用它解开资源包中的 `fleg.zip` 即可取得 flag。

决定性障碍是从游戏贴图中识别并提取隐藏信息，因此归入 `stego`。仓库含数千张普通 Minecraft 材质，但只有 `polished_basalt_top.png` 承载独立解题信息，故只提取这一张图。

![Polished Basalt 顶面贴图中替换出的可扫描二维码](bi0sCTF2025-mineworlds-wp/polished-basalt-qr.png)

## 解题过程

`Admin/Exploit/brute.py` 定义了服务器区域定位所用的哈希：取 SHA-256 摘要前 4 字节的大端整数，再清除最高符号位，目标值为 `1317778666`。

```python
import hashlib

def sha256_as_int(value: str) -> int:
    digest = hashlib.sha256(value.encode("utf-8")).digest()
    return int.from_bytes(digest[:4], "big") & 0x7fffffff

target_hash = 1317778666
salt = ":why_so_salty#LazyCrypto"

for i in range(10**10):
    candidate = f"Candidate{i}"
    if sha256_as_int(candidate + salt) == target_hash:
        print(candidate)
        break
```

官方脚本把上限写成 $10^{10}$，属于可能长时间运行的穷举；本次没有执行该步骤，也没有启动 Minecraft 服务。仓库 README 已说明其作用是得到已加载区域，随后由游戏内提示把范围缩小到约 3 至 4 个方块。该线上定位阶段在当前归档中没有保存最终坐标，所以这里不伪造坐标或运行日志。

离线材料足以复核后半条链。目标贴图位于：

```text
Admin/src/Texture_pack/assets/minecraft/textures/block/polished_basalt_top.png
```

它只有 `32x32` 像素，直接识别容易失败；用最近邻方式放大后扫描，二维码内容为：

```text
https://tinyurl.com/mc-mikey
```

短链跳转到 `https://pastebin.com/RWJxAU9u`，页面正文的有效信息已经转写如下，因此即使外链失效也不影响 WP：

```text
Wait is this being downloaded on my system?
pw: b4i2pIR8eB6u30QpCIWb0Efi
```

提示中的 “downloaded on my system” 指向客户端已经下载的资源包。仓库内对应文件为 `Admin/src/Texture_pack/fleg.zip`。它包含加密条目 `fleg.txt`，用二维码给出的口令读取：

```python
import zipfile

with zipfile.ZipFile("fleg.zip") as archive:
    flag = archive.read(
        "fleg.txt",
        pwd=b"b4i2pIR8eB6u30QpCIWb0Efi",
    )
    print(flag.decode())
```

输出为：

```text
bi0sctf{ch1ck3n_j0ckey_m1ght_b3_my_0nly_fr13nd}
```

该值与仓库 README 的官方 flag 一致。二维码、口令、加密 ZIP 与最终文本构成了可独立复核的材料级闭环；未复现的仅是服务器内行走和坐标定位。

## 方法总结

本题应把“线上定位”和“离线隐写”拆开验证：截断哈希与游戏内提示负责找到方块，重纹理的 Polished Basalt 顶面负责传递二维码，二维码口令再解开客户端资源包里的 ZIP。没有保存坐标时可以明确标注缺口，但不能省略后半段已经能够完整复核的提取链。

图片筛选也应服从解题证据：普通方块纹理、UI 图标和装饰资源没有独立信息，不应随 WP 搬运；只有二维码贴图值得以语义文件名保留。
