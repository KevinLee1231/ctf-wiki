# Login

## 题目简述

题目给出一个 Android APK 和一份 HTTP 流量。APK 的 Java 层负责发起登录请求，实际的加密和签名都在 `liblogin.so` 的 native 方法中完成；流量中的请求体是加密后的登录数据，`sign` 被放入 HTTP header。

启动流程里 `SplashActivity` 先向服务端请求 key，再调用 native `setkey` 写入全局 key。`encrypt` 会拼接：

```text
username:password:androidId
```

然后做 16 字节对齐、改版 AES 加密和改表 Base64 编码。改版 AES 的关键差异是：自定义 S-box、调换 `ShiftRows` 与 `SubBytes` 顺序、`AddRoundKey` 时额外异或 `0x91`。`sign` 则是标准 MD5，签名明文为：

```text
VNCTF:username:password:encrypted:checkIDA:checkFrida:checkMaps
```

其中后三个检查值由反调试线程维护，用于检测 `ida_server`、`frida_server` 和 maps 风险环境；任一为 0，服务端会返回 `Risky Environment`。解题需要从流量和 APK 中恢复账号、密码、androidId，再用 Frida 固定 androidId 并绕过反调试检查。

## 解题过程

附件给了一个apk和流量包
打开流量包可以发现明显的 HTTP 登录请求，body 是 Base64 后的 JSON。接着分析 APK，Java 层只负责组织请求，核心加密和签名都转入 native：

```java
String encrypted = Login.encrypt(username, password, androidId);
String sign = Login.sign(username, password, encrypted);
request.addHeader("sign", sign);
request.setBody(encrypted);
```

`SplashActivity` 会先请求服务端 key，再调用 native `setkey` 写入全局 key；`JNI_OnLoad` 注册剩余 native 方法。native 层的关键逻辑可整理为：

```c
setkey(key):
    global_key = key

encrypt(username, password):
    android_id = get_android_id()
    plain = sprintf("%s:%s:%s", username, password, android_id)
    plain = pad_to_16_bytes(plain)
    cipher = modified_aes_encrypt(plain, global_key)
    return modified_base64_encode(cipher)

sign(username, password, encrypted):
    msg = sprintf(
        "VNCTF:%s:%s:%s:%d:%d:%d",
        username, password, encrypted,
        checkIDA, checkFrida, checkMaps
    )
    return md5_hex(msg)
```

`encrypt` 中加密内容为 `username:password:androidId`，服务端会记录 `androidId` 用于验证登录设备。改版 AES 的差异集中在三处：

```text
1. 使用自定义 S-box。
2. 调换 ShiftRows 和 SubBytes 的顺序。
3. AddRoundKey 时额外异或 0x91。
```

后续 Base64 编码函数只魔改了 `base64_table`。`sign` 里的 `checkIDA/checkFrida/checkMaps` 由三个反调试线程维护，分别检测 `ida_server`、`frida_server` 和 maps 风险环境；任一为 0，服务端会直接返回 `Risky Environment`。MD5 本身是标准 MD5，没有魔改。

所以 sign 函数进行 md5 的数据为：

```text
VNCTF:username:password:encrypted:checkIDA:checkFrida:checkMaps
```

接着就可以对流量包数据直接解密

```
#include<stdio.h>
#include<stdint.h>
#include<string.h>
typedef struct
{
  uint32_t eK[44], dK[44]; // encKey, decKey
  int Nr;                  // 10 rounds
} AesKey;
```

#define BLOCKSIZE 16 // AES-128分组长度为16字节

```
// uint8_t y[4] -> uint32_t x
#define LOAD32H(x, y)
\
  do
\
  {
\
    (x) = ((uint32_t)((y)[0] & 0xff) << 24) | ((uint32_t)((y)[1] & 0xff) << 16)
| \
          ((uint32_t)((y)[2] & 0xff) << 8) | ((uint32_t)((y)[3] & 0xff));
\
} while (0)
// uint32_t x -> uint8_t y[4]
#define STORE32H(x, y)                      \
  do                                        \
  {                                         \
    (y)[0] = (uint8_t)(((x) >> 24) & 0xff); \
    (y)[1] = (uint8_t)(((x) >> 16) & 0xff); \
    (y)[2] = (uint8_t)(((x) >> 8) & 0xff);  \
    (y)[3] = (uint8_t)((x) & 0xff);         \
  } while (0)
```

// 从uint32_t x中提取从低位开始的第n个字节
#define BYTE(x, n) (((x) >> (8 * (n))) & 0xff)

```
// 密钥扩展中的SubWord(RotWord(temp),字节替换然后循环左移1位
#define MIX(x) (((S[BYTE(x, 2)] << 24) & 0xff000000) ^ ((S[BYTE(x, 1)] << 16) &
0xff0000) ^ \
                ((S[BYTE(x, 0)] << 8) & 0xff00) ^ (S[BYTE(x, 3)] & 0xff))
```

// uint32_t x循环左移n位
#define ROF32(x, n) (((x) << (n)) | ((x) >> (32 - (n))))
// uint32_t x循环右移n位
#define ROR32(x, n) (((x) >> (n)) | ((x) << (32 - (n))))

```
// AES-128轮常量,无符号长整型
static const uint32_t rcon[10] = {
    0x01000000UL, 0x02000000UL, 0x04000000UL, 0x08000000UL, 0x10000000UL,
    0x20000000UL, 0x40000000UL, 0x80000000UL, 0x1B000000UL, 0x36000000UL};
// S盒
unsigned char S[256] =
{0x20,0x7b,0x18,0xa7,0x42,0x44,0xd7,0x4a,0xcd,0x32,0xd1,0xec,0xf3,0x81,0xa5,0x8
9,

0x0e,0x91,0x4b,0xf0,0xe9,0x5d,0x8d,0xf5,0x46,0xfc,0x31,0x36,0xb6,0xac,0x9b,0xb9
,

0x26,0x09,0xe6,0x40,0xd4,0xb0,0x51,0x4f,0x9c,0x3e,0xe7,0x79,0x30,0x88,0xb1,0x3c
,

0x7a,0x5c,0xd3,0x14,0x5a,0xab,0x56,0xc0,0x04,0x29,0xd0,0x3b,0x1f,0xf9,0xa3,0x57
,

0x00,0x8a,0x84,0x16,0xf4,0x1a,0xea,0x64,0xa6,0xd6,0x2e,0xbe,0x2f,0x17,0xc4,0xe0
,

0x1e,0x02,0x3a,0x22,0x8f,0x9f,0xcb,0xa8,0x2c,0x67,0x34,0x25,0xd5,0xff,0xef,0xf6
,

0xe2,0xaa,0xd9,0x72,0xfe,0xce,0xa1,0x78,0x85,0x96,0x2a,0x77,0xca,0xc1,0x37,0x74
,

0xa2,0x5e,0x6c,0xfd,0xb8,0x4d,0x7d,0x70,0xb3,0xdd,0xcf,0x71,0x73,0x61,0xf8,0x19
,

0x48,0xe3,0x63,0x33,0x3d,0x15,0xae,0x98,0xe5,0x80,0xbd,0xbc,0x82,0xc6,0x94,0x01
,

0xe4,0xde,0x06,0x50,0x95,0xdf,0x47,0xf7,0x90,0x8b,0x45,0x9a,0x6e,0x07,0xad,0x1c
,
0x35,0x83,0x68,0x03,0x6f,0x5b,0xb7,0xfb,0x1d,0xc5,0x10,0x7c,0xd8,0x6a,0xcc,0x69
,

0x8e,0x24,0x4c,0x39,0xb4,0xa0,0x0b,0x52,0xe8,0xa9,0xb2,0x8c,0x0a,0xbf,0x28,0x86
,

0x6d,0xaf,0xda,0x41,0xfa,0x75,0xb5,0x43,0xc3,0x60,0x62,0x2b,0x55,0xf2,0x9e,0x2d
,

0x12,0x23,0x0d,0xdb,0x6b,0xc7,0x38,0x7f,0x5f,0x97,0x08,0xed,0xe1,0xbb,0xee,0x9d
,

0xd2,0x92,0x49,0x3f,0xdc,0x58,0x87,0xc2,0xba,0x99,0xc9,0x4e,0xf1,0x21,0xeb,0x13
,

0x65,0x59,0x76,0x0c,0xc8,0x05,0xa4,0x54,0x93,0x1b,0x66,0x11,0x27,0x53,0x7e,0x0f
};
// 逆S盒
unsigned char inv_S[256] = { 0 };
/* copy in[16] to state[4][4] */
int loadStateArray(uint8_t (*state)[4], const uint8_t *in)
{
  for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    {
      state[j][i] = *in++;
    }
  }
  return 0;
}
/* copy state[4][4] to out[16] */
int storeStateArray(uint8_t (*state)[4], uint8_t *out)
{
  for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    {
      *out++ = state[j][i];
    }
  }
  return 0;
}
// 密钥扩展，只接受16字初始密钥
int keyExpansion(const uint8_t *key, uint32_t keyLen, AesKey *aesKey)
{
if (NULL == key || NULL == aesKey)
  {
    printf("keyExpansion param is NULL\n");
    return -1;
  }
if (keyLen != 16)
  {
    printf("keyExpansion keyLen = %d, Not support.\n", keyLen);
    return -1;
}
uint32_t *w = aesKey->eK; // 加密密钥
  uint32_t *v = aesKey->dK; // 解密密钥
```

// 扩展密钥长度44=4*(10+1)个字,原始密钥128位，4个32位字，Nb*(Nr+1)

```
/* W[0-3],前4个字为原始密钥 */
  for (int i = 0; i < 4; ++i)
  {
    LOAD32H(w[i], key + 4 * i);
  }
/* W[4-43] */
  // temp=w[i-1];tmp=SubWord(RotWord(temp))xor Rcon[i/4] xor w[i-Nk]
  for (int i = 0; i < 10; ++i)
  {
    w[4] = w[0] ^ MIX(w[3]) ^ rcon[i];
    w[5] = w[1] ^ w[4];
    w[6] = w[2] ^ w[5];
    w[7] = w[3] ^ w[6];
    w += 4;
  }
w = aesKey->eK + 44 - 4;
  // 解密密钥矩阵为加密密钥矩阵的倒序，方便使用，把ek的11个矩阵倒序排列分配给dk作为解密密钥
  // 即dk[0-3]=ek[41-44], dk[4-7]=ek[37-40]... dk[41-44]=ek[0-3]
  for (int j = 0; j < 11; ++j)
  {
    for (int i = 0; i < 4; ++i)
    {
      v[i] = w[i];
    }
    w -= 4;
    v += 4;
  }
return 0;
}
// 轮密钥加
int addRoundKey(uint8_t (*state)[4], const uint32_t *key)
{
  uint8_t k[4][4];
/* i: row, j: col */
  for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    {
      k[i][j] = (uint8_t)BYTE(key[j], 3 - i); /* 把 uint32 key[4] 先转换为矩阵
uint8 k[4][4] */
      state[i][j] ^= k[i][j] ^ 0x91;
    }
  }
return 0;
}
// 字节替换
int subBytes(uint8_t (*state)[4])
{
  /* i: row, j: col */
  for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    {
      state[i][j] = S[state[i][j]]; // 直接使用原始字节作为S盒数据下标
    }
  }
return 0;
}
// 逆字节替换
int invSubBytes(uint8_t (*state)[4])
{
  /* i: row, j: col */
  for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    {
      state[i][j] = inv_S[state[i][j]];
    }
  }
  return 0;
}
// 行移位
int shiftRows(uint8_t (*state)[4])
{
  uint32_t block[4] = {0};
/* i: row */
  for (int i = 0; i < 4; ++i)
  {
    // 便于行循环移位，先把一行4字节拼成uint_32结构，移位后再转成独立的4个字节uint8_t
    LOAD32H(block[i], state[i]);
    block[i] = ROF32(block[i], 8 * i); // block[i]循环左移8*i位，如第0行左移0位
    STORE32H(block[i], state[i]);
  }
  return 0;
}
// 逆行移位
int invShiftRows(uint8_t (*state)[4])
{
  uint32_t block[4] = {0};
/* i: row */
  for (int i = 0; i < 4; ++i)
  {
    LOAD32H(block[i], state[i]);
    block[i] = ROR32(block[i], 8 * i);
    STORE32H(block[i], state[i]);
  }
return 0;
}
/* Galois Field (256) Multiplication of two Bytes */
// 两字节的伽罗华域乘法运算
uint8_t GMul(uint8_t u, uint8_t v)
{
  uint8_t p = 0;
for (int i = 0; i < 8; ++i)
  {
    if (u & 0x01)
    {
      p ^= v;
    }
int flag = (v & 0x80);
    v <<= 1;
    if (flag)
    {
      v ^= 0x1B;
    }
u >>= 1;
  }
return p;
}
// 列混合
int mixColumns(uint8_t (*state)[4])
{
  uint8_t tmp[4][4];
  uint8_t M[4][4] = {{0x02, 0x03, 0x01, 0x01},
                     {0x01, 0x02, 0x03, 0x01},
                     {0x01, 0x01, 0x02, 0x03},
                     {0x03, 0x01, 0x01, 0x02}};
/* copy state[4][4] to tmp[4][4] */
  for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    {
      tmp[i][j] = state[i][j];
    }
  }
for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    { // 伽罗华域加法和乘法
      state[i][j] = GMul(M[i][0], tmp[0][j]) ^ GMul(M[i][1], tmp[1][j]) ^
GMul(M[i][2], tmp[2][j]) ^ GMul(M[i][3], tmp[3][j]);
    }
  }
return 0;
}
// 逆列混合
int invMixColumns(uint8_t (*state)[4])
{
  uint8_t tmp[4][4];
  uint8_t M[4][4] = {{0x0E, 0x0B, 0x0D, 0x09},
                     {0x09, 0x0E, 0x0B, 0x0D},
                     {0x0D, 0x09, 0x0E, 0x0B},
{0x0B, 0x0D, 0x09, 0x0E}}; // 使用列混合矩阵的逆矩阵
/* copy state[4][4] to tmp[4][4] */
  for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    {
      tmp[i][j] = state[i][j];
    }
  }
for (int i = 0; i < 4; ++i)
  {
    for (int j = 0; j < 4; ++j)
    {
      state[i][j] = GMul(M[i][0], tmp[0][j]) ^ GMul(M[i][1], tmp[1][j]) ^
GMul(M[i][2], tmp[2][j]) ^ GMul(M[i][3], tmp[3][j]);
    }
  }
return 0;
}
// AES128解密， 参数要求同加密
int aesDecrypt(const uint8_t *key, uint32_t keyLen, const uint8_t *ct, uint8_t
*pt, uint32_t len)
{
  AesKey aesKey;
  uint8_t *pos = pt;
  const uint32_t *rk = aesKey.dK; // 解密密钥指针
  uint8_t out[BLOCKSIZE] = {0};
  uint8_t actualKey[16] = {0};
  uint8_t state[4][4] = {0};
if (NULL == key || NULL == ct || NULL == pt)
  {
    printf("param err.\n");
    return -1;
  }
if (keyLen > 16)
  {
    printf("keyLen must be 16.\n");
    return -1;
  }
if (len % BLOCKSIZE)
  {
    printf("inLen is invalid.\n");
    return -1;
  }
memcpy(actualKey, key, keyLen);
  keyExpansion(actualKey, 16, &aesKey); // 密钥扩展，同加密
for (int i = 0; i < len; i += BLOCKSIZE)
  {
    // 把16字节的密文转换为4x4状态矩阵来进行处理
    loadStateArray(state, ct);
    // 轮密钥加，同加密
    addRoundKey(state, rk);
for (int j = 1; j < 10; ++j)
    {
      rk += 4;
      invSubBytes(state);     // 逆字节替换，这两步顺序可以颠倒
      invShiftRows(state);    // 逆行移位
      addRoundKey(state, rk); // 轮密钥加，同加密
      invMixColumns(state);   // 逆列混合
    }
invSubBytes(state);  // 逆字节替换
    invShiftRows(state); // 逆行移位
    // 此处没有逆列混合
    addRoundKey(state, rk + 4); // 轮密钥加，同加密
storeStateArray(state, pos); // 保存明文数据
    pos += BLOCKSIZE;            // 输出数据内存指针移位分组长度
    ct += BLOCKSIZE;             // 输入数据内存指针移位分组长度
    rk = aesKey.dK;              // 恢复rk指针到秘钥初始位置
  }
  return 0;
}
int main() {
    for (int i = 0; i < 256; i++) inv_S[S[i]] = (unsigned char)i;
    char key[] = "MnpiiylSrRk_mZ-H";
    uint8_t decoded[] =
{0x6b,0xb8,0xa4,0xdd,0x01,0x6b,0x3c,0xdb,0x93,0xb4,0x53,0xab,0x8d,0x0b,0xfe,0xb
c,0x9b,0x58,0x4a,0xd7,0x63,0xfd,0x2a,0xcb,0x78,0x41,0x63,0x5e,0x5a,0xaa,0xd0,0x
aa,0x7e,0x2e,0x75,0x15,0x12,0x0a,0xa0,0x81,0x00,0xdc,0x9a,0x45,0x9d,0x8d,0x72,0
x8c};
    uint8_t decrypted[49] = {0};
    aesDecrypt((unsigned char *)key, 16, decoded, decrypted, 48);
    printf("%s", decrypted);
    return 0;
}
```

解密结果为
VNCTF2026:Vv&nN_W3lC0me!!:b2e90a5f379ea4db
接着我们需要做的就是伪造androidId来骗过服务端的设备校验
我这里直接使用frida脚本，直接登录即可

```
function hook_sprintf() {
    var base = Module.getBaseAddress("liblogin.so");
    Interceptor.attach(base.add(0x26178), {
        onEnter: function (args) {
            var format = args[3].readCString();
            if(format.indexOf("VNCTF") != -1) {
                args[5].writeUtf8String("b2e90a5f379ea4db");
                args[7] = ptr(0);
                args[8] = ptr(0);
                args[9] = ptr(0);
            } else {
                args[5].writeUtf8String("b2e90a5f379ea4db");
            }
        }
    });
}
Interceptor.attach(Module.getExportByName(null, "android_dlopen_ext"), {
    onEnter: function (args) {
this.libname = args[0].readCString();
    },
    onLeave: function (retval) {
        if (this.libname.indexOf("liblogin.so") != -1) {
            hook_sprintf();
        }
    }
})
```

## 方法总结

Android 登录逆向题要同时跟 Java 层请求拼装和 native 层算法实现。遇到“加密请求体 + header 签名 + 设备绑定”时，应先还原明文格式、加密算法和签名字符串，再处理设备 ID 与反调试。魔改 AES 往往只改 S-box、轮函数顺序或轮密钥异或常量；还原后可直接解历史流量，再用 Frida hook 参数或返回值完成设备校验伪造。
