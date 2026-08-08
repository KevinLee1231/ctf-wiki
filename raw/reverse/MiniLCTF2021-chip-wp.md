# MiniLCTF 2021 - chip

## 题目简述

程序要求输入恰好 32 字节，使用固定密钥 `deadbeef05` 调用随附件提供的 XXTEA 实现，再把 36 字节输出与内置 `encrypt_data` 比较。题名带有“chip”并不代表需要硬件接口；决定性步骤只是识别并逆向标准 XXTEA 数据格式，因此归入 Reverse。

## 解题过程

源码给出了完整加密库。该实现会把输入按小端 32 位整数打包，并在末尾额外保存原始长度；短于 16 字节的密钥会由 `FIXED_KEY` 补零到 16 字节。直接调用同一库的 `xxtea_decrypt` 可以避免自己猜测端序、补位和长度字段：

```c
#include <stdio.h>
#include <stdlib.h>
#include "xxtea.h"

int main(void) {
    unsigned char encrypt_data[] = {
        0xf3, 0x76, 0xb6, 0xa5, 0x73, 0x0e, 0x31, 0xec,
        0x5c, 0x63, 0x27, 0x06, 0x73, 0x81, 0xd4, 0x75,
        0xd9, 0xf4, 0x94, 0x80, 0xca, 0x5c, 0x6d, 0x99,
        0x54, 0x70, 0x51, 0x09, 0xd5, 0x14, 0x49, 0xf9,
        0x03, 0x59, 0xee, 0xb2
    };
    const char *key = "deadbeef05";
    size_t out_len = 0;
    unsigned char *plain = xxtea_decrypt(
        encrypt_data, sizeof(encrypt_data), key, &out_len
    );
    if (plain == NULL) {
        return 1;
    }
    printf("%.*s\n", (int)out_len, plain);
    free(plain);
    return 0;
}
```

把它与附件的 `xxtea.c` 一起编译：

```bash
gcc -O2 solve.c xxtea.c -o solve
./solve
```

输出为：

```text
MiniLCTF{hel10_hardwar3_hack3rs}
```

再把该字符串输入原程序，可满足 32 字节长度检查并产生完全相同的 36 字节密文。

## 方法总结

识别标准算法后，最稳妥的做法是复用题目实现中的解密入口，因为不同 XXTEA 库在字节序、长度尾块和密钥补零上可能不同。题目分类应看 flag 获取的决定性障碍，而不是题名：这里没有固件、总线、侧信道或硬件安全模型，属于普通程序算法恢复。
