# catfriend

## 题目简述

附件只给出一个名为 `catfriend` 的 MacOS 可执行文件。程序规模不大，函数数量约 28 个，但分析环境是 MacOS，和常见 Linux/Windows 逆向相比需要额外处理 Mach-O、ptrace 反调试和调试工具链差异。

只有一个chacha20算法，但是进行了魔改，轮数、还有QR函数，xor使用VM实现，VM也不难。考察对Chacha20算法的理解，也希望选手可以学习一种新的算法，RC4已经考烂了。反调试是很简单的ptrace，混淆也easy。

## 解题过程

解密脚本其实可以参考加密的过程，因为加密和解密都是差不多的算法，跑一遍就出来啦。

```
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include "chacha20.h"

int hex2bin(const char *hex, uint8_t *out, size_t maxlen) {
    size_t len = strlen(hex);
    size_t i;
    for (i = 0; i < len / 2 && i < maxlen; i++) {
        sscanf(hex + 2*i, "%2hhx", &out[i]);
    }
    return i;
}

int main() {
    const char *hex_cipher = "574d4354467b35613365386632622d316337642d346136662d623839652d3064336332663161346235637d";
    uint8_t cipher[64] = {0};
    size_t cipher_len = hex2bin(hex_cipher, cipher, sizeof(cipher));

    // 密钥和 nonce 全 0
    uint8_t key[32] = {0};
    uint8_t nonce[12] = {0};
    uint64_t counter = 0;

    struct chacha20_context ctx;
    chacha20_init_context(&ctx, key, nonce, counter);
    chacha20_xor(&ctx, cipher, cipher_len);

    printf("解密结果: %s\n", cipher);
    return 0;
}
```

脚本使用全 0 key、全 0 nonce 和 counter 0 初始化自定义 Chacha20 上下文，再对密文字节流执行同一套 xor 流程。由于流密码加解密同构，只要恢复题目魔改后的 Chacha20 轮函数、QR 函数和 VM 实现的 xor，复用加密流程即可解密。

脚本中使用的十六进制输入解码后对应最终结果：

```text
WMCTF{5a3e8f2b-1c7d-4a6f-b89e-0d3c2f1a4b5c}
```

## 方法总结

- 核心技巧：识别魔改 Chacha20，恢复轮数、QR 函数和 VM xor 后，利用流密码加解密同构直接跑一遍。
- 识别信号：二进制函数数量少但包含 ptrace 反调试、MacOS 目标和类似 Chacha state/quarter round 结构时，应优先还原算法而不是纠结壳。
- 复用要点：Chacha20 类流密码的关键是 keystream；key、nonce、counter 和魔改轮函数必须与题目一致。VM 只承载 xor 时，不需要完整还原复杂语义，保证输出字节一致即可。
