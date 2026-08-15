# Kyber Crystals

## 题目简述

程序用 OpenSSL 的 KEM 接口封装共享秘密，再取共享秘密前 32 字节作为 AES-256-ECB 密钥加密 flag。表面上需要攻破 Kyber 类后量子密码，但源码同时包含 `pubkey.h` 和 `privkey.h`，私钥 DER 数据也被编译进挑战程序。真正的逆向点是识别输出文件布局、提取嵌入私钥并按原流程执行解封装，而不是分析 Kyber 的数学安全性。

## 解题过程

### 确认密钥泄露与加密链

`main.c` 同时引入公钥和私钥：

```c
#include "pubkey.h"
#include "privkey.h"
```

生成密文时实际只使用公钥调用 `EVP_PKEY_encapsulate`，得到 KEM 密文 `ct` 和共享秘密 `ss`。随后程序把共享秘密的前 32 字节直接复制为 AES 密钥：

```c
unsigned char key[32] = {0};
memcpy(key, ss, 32);
EVP_EncryptInit_ex(ctx, EVP_aes_256_ecb(), NULL, key, NULL);
```

AES 使用 OpenSSL 默认填充，因此解密时也应保留 `EVP_DecryptFinal_ex` 的 PKCS#7 去填充步骤。

### 解析 `flag.enc`

输出文件没有魔数，字段按本机原生 `size_t` 直接写入：

```text
[sizeof(size_t) 字节的 ctlen]
[ctlen 字节的 KEM 密文]
[sizeof(size_t) 字节的 enc_flag_len]
[enc_flag_len 字节的 AES 密文]
```

官方二进制为 64 位，因此在同架构 C 程序中可直接用 `fread` 读取；若自行用 Python 解析，则应明确按生成端的小端 64 位无符号整数处理，不能把长度字段误当成网络字节序。

### 用嵌入私钥解封装

把 `privkey.h` 中的 DER 字节交给 OpenSSL 解码，然后对文件中的 KEM 密文调用解封装：

```c
BIO *mem = BIO_new_mem_buf(privkey_der, privkey_der_len);
EVP_PKEY *privkey = d2i_PrivateKey_bio(mem, NULL);

EVP_PKEY_CTX *ctx = EVP_PKEY_CTX_new(privkey, NULL);
EVP_PKEY_decapsulate_init(ctx, NULL);

size_t sslen = 0;
EVP_PKEY_decapsulate(ctx, NULL, &sslen, ct, ctlen);
unsigned char *ss = malloc(sslen);
EVP_PKEY_decapsulate(ctx, ss, &sslen, ct, ctlen);
```

最后以 `ss[0:32]` 为 AES-256-ECB 密钥解密尾部密文：

```c
EVP_DecryptInit_ex(aes_ctx, EVP_aes_256_ecb(), NULL, ss, NULL);
EVP_DecryptUpdate(aes_ctx, plain, &part_len, enc_flag, enc_flag_len);
EVP_DecryptFinal_ex(aes_ctx, plain + part_len, &final_len);
```

恢复出的明文为：

```text
shellmates{Kyb3r_Cry5t4l5_f0r_M4x1mum_Bl45t}
```

## 方法总结

KEM 本身没有被破解；安全边界被“把私钥一起发布并编译进程序”直接破坏。处理混合加密逆向题时，应先画清数据链：KEM 密文用于恢复共享秘密，共享秘密再派生对称密钥，真正的数据由对称算法加密。只要私钥可从源码或二进制中恢复，就应完全复现合法解密流程。同时要忠实解析程序自定义的二进制格式，尤其注意 `size_t` 的宽度、端序和 AES 填充。
