# AES-1

## 题目简述

题目给出 Base64 密文、口令 `aesiseasy` 和盐 `saltval`。源码用 PBKDF2-HMAC-SHA256 派生 256 位 AES 密钥，再通过 Java 默认的 `AES` transformation 加密。

## 解题过程

密钥派生参数必须与源码完全一致：

```text
PRF        = HMAC-SHA256
iterations = 10000
key length = 256 bits
password   = aesiseasy
salt       = saltval
```

Java 中 `Cipher.getInstance("AES")` 在常用提供者下等价于 `AES/ECB/PKCS5Padding`。先从 Base64 解码密文，再用派生密钥解密：

```java
KeySpec spec = new PBEKeySpec(password.toCharArray(), salt, 10000, 256);
SecretKey tmp = SecretKeyFactory
    .getInstance("PBKDF2WithHmacSHA256")
    .generateSecret(spec);
SecretKey key = new SecretKeySpec(tmp.getEncoded(), "AES");

Cipher cipher = Cipher.getInstance("AES");
cipher.init(Cipher.DECRYPT_MODE, key);
byte[] plain = cipher.doFinal(Base64.getDecoder().decode(ciphertext));
```

得到：

```text
n00bz{1_d0n't_l1k3_a3s_ch4ll3ng3_d0_y0u_lik3?_41703148ed8347adbe238ffbdbaf5e16}
```

## 方法总结

这是参数复现题，不是对 AES 本身的攻击。PBKDF2 的 PRF、迭代次数、盐、密钥长度以及 AES 模式和填充任一不一致，都会导致解密失败。
