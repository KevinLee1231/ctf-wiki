# N1CTF 2020 N1egg-AllSignIn Writeup

## 题目简述

这是赛事元题，不提供新的服务。README 明确要求按固定顺序拼接五道题的原始 flag，计算 MD5，再包入 `n1egg{}`：

```text
misc_signin_flag + web_signin_flag + pwn_signin_flag + re_oflo_flag + crypto_vss_flag
```

决定性步骤是严格保持原始字符串和顺序；任何大小写、花括号、换行或编码差异都会产生完全不同的摘要。

## 解题过程

### 收集五个输入

依次完成以下题目并保存平台返回的完整 flag：

1. MISC SignIn
2. Web SignIn
3. Pwn SignIn
4. RE Oflo
5. Crypto VSS

其中仓库直接给出的欢迎 flag 为：

```text
n1ctf{welc0m3_to_n1ctf2020_ctfers}
```

Web SignIn 可通过 PHP 反序列化与报错型 SQL 注入取得；Pwn SignIn 通过 vector 越界和 tcache 投毒取得；Oflo 需要去除花指令并结合 `/proc/version` 解异或；VSS 则利用二维码白边恢复 MT19937 状态。元题不改变这些子题结果，也不要求只取花括号内部。

### 按字节拼接并计算 MD5

使用二进制安全方式直接连接五个完整字符串，不插入分隔符和换行：

```python
from hashlib import md5

parts = [
    misc_signin_flag,
    web_signin_flag,
    pwn_signin_flag,
    re_oflo_flag,
    crypto_vss_flag,
]

joined = ''.join(parts).encode('utf-8')
answer = 'n1egg{' + md5(joined).hexdigest() + '}'
print(answer)
```

提交前可打印每一段的 `repr()` 和长度，确认没有从网页复制进来的 `\r`、`\n` 或不可见空格。仓库没有保存比赛平台返回的全部动态 flag，因此不能仅凭离线材料可靠重算最终摘要；这里保留可完整复现的算法，不编造最终值。

## 方法总结

元题最常见的失败不是哈希算法错误，而是输入规范错误。应明确是否保留前缀和花括号、是否有分隔符、顺序是否固定，以及摘要采用十六进制小写还是原始字节。本题 README 已把规则写成代码，最稳妥的方法就是逐字实现并对每段输入做长度检查。
