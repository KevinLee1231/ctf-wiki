# GlacierCTF2023 - sop

## 题目简述

程序把控制流拆到多个 Unix signal handler 中：`SIGSEGV`、实时信号、`SIGALRM` 等依次完成密钥设置、IV 设置、ChaCha20 轮函数、异或和比较。随机顺序比较 68 个输出字节只用于干扰分析，底层算法仍是标准 ChaCha20。

## 解题过程

沿 signal 注册和 `raise()` 链恢复状态：

- `chacha_keysetup_1` 定义完整 32 字节 key；后续 handler 通过 signal 编号 16 把指针移到后半段；
- 常量数组与循环密钥 `LMAO` 异或后得到 `expand 32-byte k`；
- `local_counter` 为 0，8 字节 nonce 是 `2f eb f0 70 5c e2 6f 80`；
- `do_round` 执行标准 20 轮 ChaCha quarter-round，`do_finish` 与固定 68 字节 target 比较。

提取 key、nonce 和 target 后，无需继续模拟 signal 控制流：

```python
from Crypto.Cipher import ChaCha20

key = bytes.fromhex(
    "17a5e693488c85f1c64bd5dc7bd8a37a"
    "1e23ce8b86c7072c8629336b8cdf2834"
)
nonce = bytes.fromhex("2febf0705ce26f80")
target = bytes.fromhex(
    "95dbcf44f84c06cb8ed60770532579a6a70bbb48cc1a84ff7d9d8707cb7bcb8e"
    "0b9035839bdf6ac5789ab2fcfce0d55135d25ac81540cc657d9066b9b8468088"
    "605748eb"
)

cipher = ChaCha20.new(key=key, nonce=nonce)
print(cipher.decrypt(target))
```

输出为：

```text
gctf{i_h4te_pr0gr4mm1ng_P4raD1gms_50_1_cRe4t3d_7h1s_fUn_ch4ll3ng3!!}
```

## 方法总结

Signal-oriented programming 让动态调试充满异步跳转，但没有改变密码算法的数据流。分析时应先记录每个 signal 的注册函数、触发顺序和共享全局状态，再把控制流噪声剥离成普通的 key/nonce/ciphertext 三元组。
