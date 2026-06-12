# Signal Storm

## 题目简述

题面提示“下个断点就能拿到 flag”，附件是本地 ELF 程序。程序表面上用 SIGSEGV、SIGTRAP、SIGFPE 等信号干扰分析，内部仍围绕 RC4 状态数组做校验：初始化 sbox 后，每个输入字符触发三类信号并调用 `swap_sbox`，随后生成 keystream 与输入异或，最终进入 `memcmp`。程序还读取 `/proc/self/status` 中的 TracerPid 并把它混入 key，属于反调试干扰。

## 解题过程

RC4 是流密码，经过 KSA 和 PRGA 生成 keystream 后与输入异或。由于 XOR 可逆，可以动态调试到 `memcmp` 前后，提取程序期望的密文和当前 keystream，再异或得到明文；也可以静态还原 keystream 生成逻辑。

程序逻辑可以概括为：`init_sbox()` 初始化 RC4 状态；每个输入字符触发 SIGSEGV、SIGTRAP、SIGFPE 三类信号；handler 中调用 `swap_sbox` 并做 byte 级 `shift_key`；随后使用当前 keystream 与输入异或。程序还读取 `/proc/self/status` 中的 `TracerPid` 并把它混入 key。如果在调试状态下直接跑，`TracerPid != 0` 会导致 keystream 错误，所以需要 patch 掉反调试逻辑，或者在无调试环境下复现算法。

解法：
- [法一：用 CyberChef 做 XOR](https://gchq.github.io/CyberChef/#recipe=From_Hex('Auto')XOR(%7B'option':'Latin1','string':'a'%7D,'Standard',false)To_Hex('0x%20with%20comma',0)&input=MHhlYSwgMHgzMCwgMHhkOSwgMHhjNCwgMHhjZCwgMHhkYiwgMHg0ZiwgMHg5NiwgMHg3OCwgMHhkNCwgMHgyZiwgMHgyNywgMHg4NCwgMHhiZiwgMHg0NywgMHgyMiwgMHg1YiwgMHhlZSwgMHgzYiwgMHg3OCwgMHgxMiwgMHhiYiwgMHg5LCAweGNmLCAweDU3LCAweDliLCAweGUsIDB4MzMsIDB4NzEsIDB4ZTUsIDB4NTYsIDB4Ng&oeol=FF)
- 法二：还原生成 keystream 的逻辑

```c
#include <stdio.h>
#include <string.h>

unsigned char enc[] = {0xe3, 0x36, 0xd9, 0xc8, 0xc9, 0xc1, 0x60, 0x82,
                       0x75, 0xd9, 0x11, 0x25, 0xd5, 0xb2, 0x4b, 0x1c,
                       0x4d, 0xe6, 0x6d, 0x71, 0x1c, 0xaf, 0x1c, 0xf1,
                       0x06, 0xa5, 0x1c, 0x26, 0x7f, 0xf6, 0x5a, 0x1a};
unsigned char S[256], k[] = "C0lm_be4ore_7he_st0rm";
int i, j, t, ig = 0, jg = 0;

int main() {
  for (i = 0; i < 256; i++)
    S[i] = i;
  for (i = j = 0; i < 256; i++) {
    j = (j + S[i] + k[i % 21]) % 256;
    t = S[i];
    S[i] = S[j];
    S[j] = t;
  }
  for (int x = 0; x < sizeof(enc); x++) {
    ig = (ig + 1) % 256;
    jg = (jg + S[ig] + k[ig % 21]) % 256;
    t = S[ig];
    S[ig] = S[jg];
    S[jg] = t;
    unsigned char f = k[0];
    memmove(k, k + 1, 20);
    k[20] = f;
    putchar(enc[x] ^ S[(S[ig] + S[jg]) % 256]);
  }
}
// hgame{Null_c0lm_wi7hout_0_storm}
```

## 方法总结

信号混淆不要只看异常表面，要还原每个 handler 对状态机的实际影响。只要最终校验是 XOR/RC4 一类可逆流加密，动态断在 `memcmp` 或比较前后提取 keystream，往往比静态完整还原更快。
