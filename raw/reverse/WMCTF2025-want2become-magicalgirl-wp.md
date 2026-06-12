# Want2BecomeMagicalGirl

## 题目简述

题目拥有两个frida检测点，一个在Java层，一个在flutter层，Java层检测Frida端口，flutter检测Java层是否被hook。

附件只给出 `Want2BecomeMagicalGirl.apk`。这是 Android Flutter、Java 和 native 混合逆向题，Flutter 层负责输入和最终判断，Java/native 层负责中间加密与反调试/反 hook，不能只静态看 Dart 或 Java 某一层。

题目还有一个简单加壳考点，可以称为抽取壳：`nativeadd` 的 `.init_array` 中有一段程序实现 self-hook，分别 hook 了 `libart.so` 的 `art::interpreter::DoCall<false,false>(art::ClassLinker *this)` 和 `art::OatHeader::IsDebuggable(art::OatHeader *__hidden this)`。

这两个 hook 会影响 ART 解释执行和 debuggable 判断，并进一步改变 Java 层 XXTEA 的真实执行流程。直接静态看 Java 代码容易得到错误结论，推荐用 smali trace 跟真实执行路径。

## 解题过程

### 执行流程与算法还原

具体程序执行流程是：Flutter 输入 -> 加载 `libnative.so` -> 修改字节码 -> 判断 `libart.so` 指定位置有没有被修改 -> 魔改 AES 加密 -> 魔改 Java 加密 -> Flutter 判断密文。
在Flutter层只有一个aes加密，很常规地魔改了sbox以及_addRoundKey()和_mixColumns()的执行顺序。
在Java层魔改了xxtea的执行流程把所有`<<` 变为了`>>`，同时还魔改了轮加密轮数改为`rounds = 6 + 52 * n`。
以下是解题脚本：

```c
#include <inttypes.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// base64 加密
char base64[65] =
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

int decodeBase64(char *str, int len, char **in) {

  char ascill[129];
  int k = 0;
  for (int i = 0; i < 64; i++) {
    ascill[base64[i]] = k++;
  }
  int decodeStrlen = len / 4 * 3 + 1;
  char *decodeStr = (char *)malloc(sizeof(char) * decodeStrlen);
  k = 0;
  for (int i = 0; i < len; i++) {
    decodeStr[k++] = (ascill[str[i]] << 2) | (ascill[str[++i]] >> 4);
    if (str[i + 1] == '=') {
      break;
    }
    decodeStr[k++] = (ascill[str[i]] << 4) | (ascill[str[++i]] >> 2);
    if (str[i + 1] == '=') {
      break;
    }
    decodeStr[k++] = (ascill[str[i]] << 6) | (ascill[str[++i]]);
  }
  decodeStr[k] = '\0';
  *in = decodeStr;
  return k;
}
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
void AddRoundKey(unsigned char *plaintext,
                 unsigned char *CipherKey)
{
  for (int j = 0; j < 16; j++)
    plaintext[j] = plaintext[j] ^ CipherKey[j];
}
void SubBytes(unsigned char *plaintext, unsigned char *plaintextencrypt,
              int count)
{
  unsigned int row, column;
  unsigned char Sbox[16][16] = {
      0x20, 0x7b, 0x18, 0xa7, 0x42, 0x44, 0xd7, 0x4a, 0xcd, 0x32, 0xd1, 0xec,
      0xf3, 0x81, 0xa5, 0x89, 0x0e, 0x91, 0x4b, 0xf0, 0xe9, 0x5d, 0x8d, 0xf5,
      0x46, 0xfc, 0x31, 0x36, 0xb6, 0xac, 0x9b, 0xb9, 0x26, 0x09, 0xe6, 0x40,
      0xd4, 0xb0, 0x51, 0x4f, 0x9c, 0x3e, 0xe7, 0x79, 0x30, 0x88, 0xb1, 0x3c,
      0x7a, 0x5c, 0xd3, 0x14, 0x5a, 0xab, 0x56, 0xc0, 0x04, 0x29, 0xd0, 0x3b,
      0x1f, 0xf9, 0xa3, 0x57, 0x00, 0x8a, 0x84, 0x16, 0xf4, 0x1a, 0xea, 0x64,
      0xa6, 0xd6, 0x2e, 0xbe, 0x2f, 0x17, 0xc4, 0xe0, 0x1e, 0x02, 0x3a, 0x22,
      0x8f, 0x9f, 0xcb, 0xa8, 0x2c, 0x67, 0x34, 0x25, 0xd5, 0xff, 0xef, 0xf6,
      0xe2, 0xaa, 0xd9, 0x72, 0xfe, 0xce, 0xa1, 0x78, 0x85, 0x96, 0x2a, 0x77,
      0xca, 0xc1, 0x37, 0x74, 0xa2, 0x5e, 0x6c, 0xfd, 0xb8, 0x4d, 0x7d, 0x70,
      0xb3, 0xdd, 0xcf, 0x71, 0x73, 0x61, 0xf8, 0x19, 0x48, 0xe3, 0x63, 0x33,
      0x3d, 0x15, 0xae, 0x98, 0xe5, 0x80, 0xbd, 0xbc, 0x82, 0xc6, 0x94, 0x01,
      0xe4, 0xde, 0x06, 0x50, 0x95, 0xdf, 0x47, 0xf7, 0x90, 0x8b, 0x45, 0x9a,
      0x6e, 0x07, 0xad, 0x1c, 0x35, 0x83, 0x68, 0x03, 0x6f, 0x5b, 0xb7, 0xfb,
      0x1d, 0xc5, 0x10, 0x7c, 0xd8, 0x6a, 0xcc, 0x69, 0x8e, 0x24, 0x4c, 0x39,
      0xb4, 0xa0, 0x0b, 0x52, 0xe8, 0xa9, 0xb2, 0x8c, 0x0a, 0xbf, 0x28, 0x86,
      0x6d, 0xaf, 0xda, 0x41, 0xfa, 0x75, 0xb5, 0x43, 0xc3, 0x60, 0x62, 0x2b,
      0x55, 0xf2, 0x9e, 0x2d, 0x12, 0x23, 0x0d, 0xdb, 0x6b, 0xc7, 0x38, 0x7f,
      0x5f, 0x97, 0x08, 0xed, 0xe1, 0xbb, 0xee, 0x9d, 0xd2, 0x92, 0x49, 0x3f,
      0xdc, 0x58, 0x87, 0xc2, 0xba, 0x99, 0xc9, 0x4e, 0xf1, 0x21, 0xeb, 0x13,
      0x65, 0x59, 0x76, 0x0c, 0xc8, 0x05, 0xa4, 0x54, 0x93, 0x1b, 0x66, 0x11,
      0x27, 0x53, 0x7e, 0x0f};
  for (int i = 0; i < count; i++) {
    row = (plaintext[i] & 0xF0) >> 4;
    column = plaintext[i] & 0x0F;
    plaintextencrypt[i] = Sbox[row][column];
  }
}
void SubBytesRe(unsigned char *plaintext, unsigned char *plaintextencrypt,
                int count)
{
  unsigned int row, column;
  unsigned char Sbox[16][16] = {
      0x40, 0x8f, 0x51, 0xa3, 0x38, 0xf5, 0x92, 0x9d, 0xda, 0x21, 0xbc, 0xb6,
      0xf3, 0xd2, 0x10, 0xff, 0xaa, 0xfb, 0xd0, 0xef, 0x33, 0x85, 0x43, 0x4d,
      0x02, 0x7f, 0x45, 0xf9, 0x9f, 0xa8, 0x50, 0x3c, 0x00, 0xed, 0x53, 0xd1,
      0xb1, 0x5b, 0x20, 0xfc, 0xbe, 0x39, 0x6a, 0xcb, 0x58, 0xcf, 0x4a, 0x4c,
      0x2c, 0x1a, 0x09, 0x83, 0x5a, 0xa0, 0x1b, 0x6e, 0xd6, 0xb3, 0x52, 0x3b,
      0x2f, 0x84, 0x29, 0xe3, 0x23, 0xc3, 0x04, 0xc7, 0x05, 0x9a, 0x18, 0x96,
      0x80, 0xe2, 0x07, 0x12, 0xb2, 0x75, 0xeb, 0x27, 0x93, 0x26, 0xb7, 0xfd,
      0xf7, 0xcc, 0x36, 0x3f, 0xe5, 0xf1, 0x34, 0xa5, 0x31, 0x15, 0x71, 0xd8,
      0xc9, 0x7d, 0xca, 0x82, 0x47, 0xf0, 0xfa, 0x59, 0xa2, 0xaf, 0xad, 0xd4,
      0x72, 0xc0, 0x9c, 0xa4, 0x77, 0x7b, 0x63, 0x7c, 0x6f, 0xc5, 0xf2, 0x6b,
      0x67, 0x2b, 0x30, 0x01, 0xab, 0x76, 0xfe, 0xd7, 0x89, 0x0d, 0x8c, 0xa1,
      0x42, 0x68, 0xbf, 0xe6, 0x2d, 0x0f, 0x41, 0x99, 0xbb, 0x16, 0xb0, 0x54,
      0x98, 0x11, 0xe1, 0xf8, 0x8e, 0x94, 0x69, 0xd9, 0x87, 0xe9, 0x9b, 0x1e,
      0x28, 0xdf, 0xce, 0x55, 0xb5, 0x66, 0x70, 0x3e, 0xf6, 0x0e, 0x48, 0x03,
      0x57, 0xb9, 0x61, 0x35, 0x1d, 0x9e, 0x86, 0xc1, 0x25, 0x2e, 0xba, 0x78,
      0xb4, 0xc6, 0x1c, 0xa6, 0x74, 0x1f, 0xe8, 0xdd, 0x8b, 0x8a, 0x4b, 0xbd,
      0x37, 0x6d, 0xe7, 0xc8, 0x4e, 0xa9, 0x8d, 0xd5, 0xf4, 0xea, 0x6c, 0x56,
      0xae, 0x08, 0x65, 0x7a, 0x3a, 0x0a, 0xe0, 0x32, 0x24, 0x5c, 0x49, 0x06,
      0xac, 0x62, 0xc2, 0xd3, 0xe4, 0x79, 0x91, 0x95, 0x4f, 0xdc, 0x60, 0x81,
      0x90, 0x88, 0x22, 0x2a, 0xb8, 0x14, 0x46, 0xee, 0x0b, 0xdb, 0xde, 0x5e,
      0x13, 0xec, 0xcd, 0x0c, 0x44, 0x17, 0x5f, 0x97, 0x7e, 0x3d, 0xc4, 0xa7,
      0x19, 0x73, 0x64, 0x5d,
  };
  for (int i = 0; i < count; i++) {
    row = (plaintext[i] & 0xF0) >> 4;
    column = plaintext[i] & 0x0F;
    plaintextencrypt[i] = Sbox[row][column];
  }
}
void ShiftRowsRe(unsigned char *plaintextencrypt)
{
  unsigned char temp = 0;
  for (int i = 0; i < 4; i++)
  {
    for (int j = 0; j < 4 - i; j++)
    {
      temp = plaintextencrypt[i];
      for (int k = 0; k < 4; k++)
        plaintextencrypt[i + 4 * k] = plaintextencrypt[i + 4 * (k + 1)];
      plaintextencrypt[i + 12] = temp;
    }
  }
}
void ShiftRows(unsigned char *plaintextencrypt)
{
  unsigned char temp = 0;
  for (int i = 0; i < 4; i++)
  {
    for (int j = 0; j < i; j++)
    {
      temp = plaintextencrypt[i];
      for (int k = 0; k < 4; k++)
        plaintextencrypt[i + 4 * k] = plaintextencrypt[i + 4 * (k + 1)];
      plaintextencrypt[i + 12] = temp;
    }
  }
}
unsigned char Mult2(unsigned char num)
{
  unsigned char temp = num << 1;
  if ((num >> 7) & 0x01)
    temp = temp ^ 27;
  return temp;
}
unsigned char Mult3(unsigned char num) { return Mult2(num) ^ num; }
void MixColumns(unsigned char *plaintextencrypt,
                unsigned char *plaintextcrypt) {
  int i;
  for (i = 0; i < 4; i++)
    plaintextcrypt[4 * i] =
        Mult2(plaintextencrypt[4 * i]) ^ Mult3(plaintextencrypt[4 * i + 1]) ^
        plaintextencrypt[4 * i + 2] ^ plaintextencrypt[4 * i + 3];
  for (i = 0; i < 4; i++)
    plaintextcrypt[4 * i + 1] =
        plaintextencrypt[4 * i] ^ Mult2(plaintextencrypt[4 * i + 1]) ^
        Mult3(plaintextencrypt[4 * i + 2]) ^ plaintextencrypt[4 * i + 3];
  for (i = 0; i < 4; i++)
    plaintextcrypt[4 * i + 2] =
        plaintextencrypt[4 * i] ^ plaintextencrypt[4 * i + 1] ^
        Mult2(plaintextencrypt[4 * i + 2]) ^ Mult3(plaintextencrypt[4 * i + 3]);
  for (i = 0; i < 4; i++)
    plaintextcrypt[4 * i + 3] =
        Mult3(plaintextencrypt[4 * i]) ^ plaintextencrypt[4 * i + 1] ^
        plaintextencrypt[4 * i + 2] ^ Mult2(plaintextencrypt[4 * i + 3]);
}

#define xtime(x) ((x << 1) ^ (((x >> 7) & 1) * 0x1b))
#define Multiply(x, y)                                                         \
  (((y & 1) * x) ^ ((y >> 1 & 1) * xtime(x)) ^                                 \
   ((y >> 2 & 1) * xtime(xtime(x))) ^                                          \
   ((y >> 3 & 1) * xtime(xtime(xtime(x)))) ^                                   \
   ((y >> 4 & 1) * xtime(xtime(xtime(xtime(x))))))
void MixColumnsRe(unsigned char *state) {

  unsigned char a, b, c, d;
  for (int i = 0; i < 4; i++) {
    a = state[4 * i];
    b = state[4 * i + 1];
    c = state[4 * i + 2];
    d = state[4 * i + 3];
    state[4 * i] = Multiply(a, 0x0e) ^ Multiply(b, 0x0b) ^ Multiply(c, 0x0d) ^
                   Multiply(d, 0x09);
    state[4 * i + 1] = Multiply(a, 0x09) ^ Multiply(b, 0x0e) ^
                       Multiply(c, 0x0b) ^ Multiply(d, 0x0d);
    state[4 * i + 2] = Multiply(a, 0x0d) ^ Multiply(b, 0x09) ^
                       Multiply(c, 0x0e) ^ Multiply(d, 0x0b);
    state[4 * i + 3] = Multiply(a, 0x0b) ^ Multiply(b, 0x0d) ^
                       Multiply(c, 0x09) ^ Multiply(d, 0x0e);
  }
}
int CharToWord(unsigned char *character, int first)
{
  return (((int)character[first] & 0x000000ff) << 24) |
         (((int)character[first + 1] & 0x000000ff) << 16) |
         (((int)character[first + 2] & 0x000000ff) << 8) |
         ((int)character[first + 3] & 0x000000ff);
}
void WordToChar(unsigned int word, unsigned char *character)
{
  for (int i = 0; i < 4; character[i++] = (word >> (8 * (3 - i))) & 0xFF)
    ;
}
void ExtendCipherKey(unsigned int *CipherKey_word, int round)
{
  unsigned char CipherKeyChar[4] = {0}, CipherKeyCharEncrypt[4] = {0};
  unsigned int Rcon[10] = {0x01000000, 0x02000000, 0x04000000, 0x08000000,
                           0x10000000, 0x20000000, 0x40000000, 0x80000000,
                           0x1B000000, 0x36000000};
  for (int i = 4; i < 8; i++) {
    if (!(i % 4)) {
      WordToChar((CipherKey_word[i - 1] >> 24) | (CipherKey_word[i - 1] << 8),
                 CipherKeyChar);
      SubBytes(CipherKeyChar, CipherKeyCharEncrypt, 4);
      CipherKey_word[i] = CipherKey_word[i - 4] ^
                          CharToWord(CipherKeyCharEncrypt, 0) ^ Rcon[round];
    } else
      CipherKey_word[i] = CipherKey_word[i - 4] ^ CipherKey_word[i - 1];
  }
}
#include <stdint.h>
#define DELTA 0x9e3779b9
#define MX                                                                     \
  (((z >> 5 ^ y >> 2) + (y >> 3 ^ z >> 4)) ^                                   \
   ((sum ^ y) + (key[(p & 3) ^ e] ^ z)))
void btea(uint32_t *v, int n, uint32_t const key[4]) {

  uint32_t y, z, sum;
  unsigned p, rounds, e;
  for (int i = 0; i < n; i++) {
    printf("v:%d\n", v[i]);
  }
  for (int i = 0; i < 4; i++) {
    printf("k:%d\n", key[i]);
  }
  if (n > 1)
  /* Coding Part */
  {
    rounds = 6 + 52 * n;
    sum = 0;
    z = v[n - 1];
    do {
      sum += DELTA;
      e = (sum >> 2) & 3;
      for (p = 0; p < n - 1; p++) {
        y = v[p + 1];
        z = v[p] += MX;
        printf("z:%d\n", MX);
      }
      y = v[0];
      z = v[n - 1] += MX;

    } while (--rounds);
  } else if (n < -1)
  /* Decoding Part */
  {
    n = -n;
    rounds = 6 + 52 * n;
    sum = rounds * DELTA;
    y = v[0];
    do {
      e = (sum >> 2) & 3;
      for (p = n - 1; p > 0; p--) {
        z = v[p - 1];
        y = v[p] -= MX;
      }
      z = v[n - 1];
      y = v[0] -= MX;
      sum -= DELTA;
    } while (--rounds);
  }
}
void decAES(unsigned char *PlainText) {
  int i = 0, k;
  unsigned char CipherKey[16] = {122, 37, 197, 36,  198, 51, 76, 48,
                                 243, 98, 175, 172, 63,  35, 3,  213},
                CipherKey1[16] = {122, 37, 197, 36,  198, 51, 76, 48,
                                  243, 98, 175, 172, 63,  35, 3,  213},
                PlainText1[16] = {0};
  memcpy(PlainText1, PlainText, 16);

  unsigned int CipherKey_word[44] = {0};
  for (i = 0; i < 4; CipherKey_word[i++] = CharToWord(CipherKey, 4 * i))
    ;
  for (int i = 0; i < 10; i++) {
    ExtendCipherKey(CipherKey_word + 4 * i, i);
    for (k = 0; k < 4;
         WordToChar(CipherKey_word[k + 4 * (i + 1)], CipherKey + 4 * k), k++)
      ;
  }

  AddRoundKey(PlainText1, CipherKey);

  for (i = 0; i < 9; i++) {
    SubBytesRe(PlainText1, PlainText, 16);
    for (k = 0; k < 4;
         WordToChar(CipherKey_word[k + 40 - 4 * (i + 1)], CipherKey + 4 * k),
        k++)
      ;

    ShiftRowsRe(PlainText);
    MixColumnsRe(PlainText);
    AddRoundKey(PlainText, CipherKey);

    for (k = 0; k < 16; PlainText1[k] = PlainText[k], k++)
      ;
  }
  ShiftRowsRe(PlainText);
  SubBytesRe(PlainText, PlainText1, 16);
  AddRoundKey(PlainText1, CipherKey1);
  memcpy(PlainText, PlainText1, 16);
}
int hexToChar(const char *hex, unsigned char *output) {
  size_t len = strlen(hex);
  if (len % 2 != 0) {
    return -1;
  }

  for (size_t i = 0; i < len; i += 2) {
    sscanf(hex + i, "%2hhx", &output[i / 2]);
  }

  return len / 2;
}
unsigned char key[16] = "16929";
unsigned char mm[] =
    "8sAFX45zT7uc0vSUyFNNly1h/d5zTt89tV3kcVr5P5n7lRKPyYtxg31zYNB2lPV0c5nf/x2/"
    "IK94XV9Ufs9XfaDG5IXxMlZy+Z2nE+ZZRFBSpMoKzQXfUq2TSjJJfQxV";


int main() {

  char *mm2;

  int len = decodeBase64(mm, strlen(mm), &mm2);

  printf("base64decode len: %d\n", strlen(mm2));

  printf("xxtea block: %d\n", -((len >> 2) + ((len & 3) != 0)));
  btea((uint32_t *)mm2, -((len >> 2) + ((len & 3) != 0)), (uint32_t *)key);
  printf("xxtea decode: %s \n",mm2);
  unsigned char mm3[100] = {};
  int len2 = strlen(mm2) / 2;
  hexToChar(mm2, mm3);
  for (int i = 0; i < len2; i += 16) {
    decAES(&mm3[i]);
  }

  puts(mm3);
}
```

脚本按实际校验逆序恢复：先 base64 解码 Flutter 层保存的密文，再按魔改 XXTEA 解密得到十六进制串，最后把十六进制转字节并执行魔改 AES 逆过程。魔改 AES 的关键在于自定义 S-box、逆 S-box，以及 `_addRoundKey()` 和 `_mixColumns()` 顺序被调整；魔改 XXTEA 的关键在于移位方向从 `<<` 改成 `>>`，轮数改为 `6 + 52 * n`。

## 方法总结

- 核心技巧：Android Flutter + Java + native 混合逆向，绕过 Frida 检测和 libart self-hook 后，用 smali trace 确认真实加密流程，再还原魔改 AES 与魔改 XXTEA。
- 识别信号：题目同时检测 Frida 端口、Java hook 状态和 libart 关键函数是否被改写时，应警惕 native 层 self-hook 会改变 Java 层算法语义。
- 复用要点：Flutter 层、Java 层、native 层的算法顺序要按真实执行流串起来；遇到 release/oat 导致 hook 困难时，smali trace 比纯动态 hook 更稳定。恢复脚本中应保留魔改常量、轮数和运算方向，避免只写“AES+XXTEA”。
