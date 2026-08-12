# DownUnderCTF 2022 click-the-flag Writeup

## 题目简述

题目提供一个 Android 点击游戏。应用每轮生成真假旗帜，点中真旗可增加分数；界面暗示需要完成大量轮次甚至付费升级。真正的 flag 并不依赖完成游戏，而是由 APK 中的原生库从图片资源的指定字节位置逐字符取出。

## 解题过程

用 JADX 查看 `MainActivity`，可以看到应用启动时读取资源 `flag_img.png` 的前 `0x8000` 字节并传给 JNI 函数 `init(byte[])`。每轮结束会调用 `ur()`，其第二个返回值被转成字符追加到界面文本。游戏分数、真假旗选择和字符释放分别由 `us()`、`gcb()`、`ur()` 管理，因此可以直接逆向 `libchall.so`，不必自动点击 85 轮。

![APK 中既作为游戏旗帜素材、又作为隐藏字符载体的绿色旗帜图片](./DownUnderCTF2022-click-the-flag-wp/flag-sprite.png)

以 x86-64 版本的 `libchall.so` 为例，`init` 会把图片前 `0x8000` 字节复制到原生堆缓冲区。`ur` 则从 `.rodata` 中依次读取 16 位小端偏移，在图片缓冲区取一个字节后异或 `0x42`。偏移表位于文件偏移 `0x2c450`，共 85 项；开头的 `d1 6e 59 6f dc 58` 分别对应十进制位置 28369、28505、22748。

可以直接从原生库取出偏移表并恢复字符串：

```python
import struct

with open('libchall.so', 'rb') as f:
    f.seek(0x2c450)  # x86_64 APK 版本中的偏移表
    positions = struct.unpack('<85H', f.read(85 * 2))

with open('flag_img.png', 'rb') as f:
    image = f.read()

flag = bytes(image[p] ^ 0x42 for p in positions)
print(flag.decode())
```

输出为：

```text
DUCTF{d1d_y0u_r43lly_th1nk_y0u_w0uld_g3t_4_fl4g_f0r_pl4y1ng_a_game?_6e927fd2e11abcd4}
```

## 方法总结

Java 层只负责游戏流程和显示，关键数据路径位于 JNI：图片资源被复制到原生缓冲区，硬编码的 16 位位置表决定取哪些字节，统一异或 `0x42` 后形成 flag。逆向时沿 `init` 到 `ur` 的数据流追踪，比分析随机出旗或编写点击器更直接；同时要选择与偏移地址对应的同一 ABI 版本原生库。
