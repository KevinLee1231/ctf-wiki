# week3证取单简

## 题目简述

附件是 Windows 7 内存镜像。需要从系统版本、进程、命令历史、登录凭据和记事本文本中串联证据：命令历史提示用户重复使用开机密码，内存中的记事本保存了一段倒序的 OpenSSL AES 密文。恢复密码并把密文倒转后即可解密 flag。

## 解题过程

先识别镜像配置：

```bash
volatility -f mem imageinfo
```

关键输出表明首选配置为 `Win7SP1x64`，镜像记录时间为 2022-07-27 08:04:37 UTC：

```text
Suggested Profile(s): Win7SP1x64, Win7SP0x64, Win2008R2SP0x64, ...
Image date and time   : 2022-07-27 08:04:37 UTC+0000
Image local date and time: 2022-07-27 16:04:37 +0800
```

指定该 profile 枚举进程：

```bash
volatility -f mem --profile=Win7SP1x64 pslist
```

列表中存在 `cmd.exe`、`notepad.exe`、`explorer.exe` 等交互进程。进一步检查命令历史和编辑框：

```bash
volatility -f mem --profile=Win7SP1x64 cmdscan
volatility -f mem --profile=Win7SP1x64 editbox
volatility -f mem --profile=Win7SP1x64 iehistory
```

`cmdscan` 恢复出的关键命令行提示为：

```text
The author likes to use the same password, including the boot password
```

也就是作者会重复使用密码，包括开机密码。用 `hashdump` 提取本地账户 NTLM 摘要并进行字典恢复，可得到用户 `zysgmzb` 的密码：

```text
0xGame2022
```

`editbox` 从记事本控件中恢复出 64 字符密文：

```text
St1gvdn13d2SGcKvxRq4vbGEKf66e1IX1ywid5epVjAHknLqo5UQj/1XkVGdsF2U
```

题目名“证取单简”是“简单取证”的倒序，提示把字符串反转，得到：

```text
U2FsdGVkX1/jQU5oqLnkHAjVpe5diwy1XI1e66fKEGbv4qRxvKcGS2d31ndvg1tS
```

前缀 `U2FsdGVkX1` 经 Base64 解码是 `Salted__`，表明它是 OpenSSL 的带盐封装。该样本使用 AES-256-CBC 和旧式 MD5 `EVP_BytesToKey` 派生，直接用恢复的密码解密：

```bash
printf '%s' 'U2FsdGVkX1/jQU5oqLnkHAjVpe5diwy1XI1e66fKEGbv4qRxvKcGS2d31ndvg1tS' \
  | openssl enc -d -aes-256-cbc -a -A -md md5 \
      -pass pass:0xGame2022
```

输出：

```text
0xGame{F1rst_St3p_0f_Forens1cs}
```

## 方法总结

内存取证的重点是证据关联，而不是孤立运行插件：`imageinfo` 确定 profile，`pslist` 指向交互进程，`cmdscan` 给出密码复用线索，凭据提取恢复密码，`editbox` 找到密文，题目名再提示倒序。`U2FsdGVkX1` 只说明 OpenSSL `Salted__` 格式，并不等于“随便找一个 AES 网站”；记录算法、KDF、摘要和密码后才能离线稳定复现。
