# DownUnderCTF 2020 - 16 Home Runs

## 题目简述

题面给出字符串 `RFVDVEZ7MTZfaDBtM19ydW41X20zNG41X3J1bm4xbjZfcDQ1N182NF9iNDUzNX0=`，标题把 “base” 与 “64” 分别包装成棒球的垒与全垒打数量提示。字符集由大小写字母、数字和末尾 `=` 填充组成，符合标准 Base64 表示层编码。

## 解题过程

直接使用标准 Base64 解码，不需要密钥：

```bash
printf '%s' 'RFVDVEZ7MTZfaDBtM19ydW41X20zNG41X3J1bm4xbjZfcDQ1N182NF9iNDUzNX0=' | base64 -d
```

输出为：

```text
DUCTF{16_h0m3_run5_m34n5_runn1n6_p457_64_b4535}
```

`=` 只是把编码长度补齐到 4 的倍数，并不是加密产生的密文特征。

## 方法总结

- 核心技巧：识别并解码标准 Base64。
- 识别信号：字符串只使用 Base64 字符表，长度通常是 4 的倍数，末尾可能有一个或两个 `=`。
- 复用要点：Base64 是可逆表示而非加密；先做格式和长度验证，再用本地工具解码，没必要把未知内容提交给在线站点。
