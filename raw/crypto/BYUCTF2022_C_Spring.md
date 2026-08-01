# BYUCTF 2022 - C-Spring

## 题目简述

选手拿到 Go 可执行文件及一组 Base64 编码的 nonce、密文。程序使用 AES-GCM，但密钥与 nonce 都来自以当前 Unix 秒为种子的 `math/rand`；决定安全性的随机源是可预测的。

## 解题过程

逆向二进制可恢复以下生成顺序：

```go
rand.Seed(time.Now().Unix())
key := make([]byte, 16)
rand.Read(key)
nonce := make([]byte, 12)
rand.Read(nonce)
```

题目公开了 nonce，因此它就是验证种子的强判据。比赛在线时可从当前 Unix 时间向前枚举；多年后的归档复现应从 `output.txt`、二进制构建记录或 BYUCTF 2022 比赛窗口附近开始，不能从今天盲扫数年秒数。每个候选种子必须先生成 16 字节 key，再生成 12 字节 nonce，不能只生成 nonce。候选 nonce 与公开值一致时，同一轮得到的 key 即为正确 AES-128 密钥。

随后以公开 nonce 调用 AES-GCM 解密并校验认证标签：

```go
block, _ := aes.NewCipher(key)
gcm, _ := cipher.NewGCM(block)
plain, err := gcm.Open(nil, realNonce, ciphertext, nil)
```

认证成功后得到：

```text
byuctf{dont_seed_with_time_in_secure_contexts_WiOuYiaZ}
```

## 方法总结

AES-GCM 本身没有被攻破；漏洞是用秒级时间种子驱动非密码学 PRNG。已知 nonce 同时提供种子验证，GCM 标签又提供最终正确性验证，使时间窗口枚举十分可靠。
