# DownUnderCTF 2022 pac Writeup

## 题目简述

内核模块 `/dev/pac` 实现了一套自制 pointer authentication。模块启动时随机生成 32 位 `KEY1`，把函数指针 `hello` 编码后通过 `read` 泄漏；向设备 `write` 一个通过认证的指针，模块就把它当作函数调用。内核启用 KASLR，却没有启用 SMEP/SMAP。

编码结果的低 32 位仍是原指针低位，高 32 位是自制分组变换产生的 tag：

$$
\operatorname{PAC}(p)=\bigl(E_{K,p}(p)\ll32\bigr)\mathbin{|}p,
\qquad p=\operatorname{ptr}\bmod 2^{32}.
$$

因此 `enc_hello` 同时泄漏了一个已知明文低 32 位和对应 tag，可用于恢复仅 32 位的认证密钥。

## 解题过程

### 把自制密码化为有限域方程

算法把 32 位状态拆成 8 个 nibble，进行 63 轮 S-box、P-box 和 round key 异或。round key 为：

$$
k_{8j+i}=K_1\oplus\left(j\cdot RC_i\cdot p\bmod 2^{32}\right).
$$

给定泄漏中的 $p$ 后，除 $K_1$ 外全是常量。更致命的是题目 S-box 在

$$
\operatorname{GF}(2^4)=\operatorname{GF}(2)[x]/(x^4+x+1)
$$

上只是仿射映射：

$$
S(z)=(\alpha+1)z+\alpha^3.
$$

P-box 只是 nibble 置换，异或也是线性运算。因此整套 tag 计算可写成关于 8 个密钥 nibble $k_0,\ldots,k_7$ 的仿射多项式组。把 `enc_hello` 的低 32 位作为 $p$、高 32 位作为目标密文，在 Sage 中建立 8 个等式并求 ideal 的 variety，即可得到唯一 `KEY1`。

### 为用户态提权函数伪造认证指针

静态 exploit 中的 `privesc` 地址低于 `0x00500000`。用恢复的 `KEY1` 对其低 32 位运行同一算法，拼成：

```text
(computed_tag << 32) | low32(privesc)
```

模块验证通过后，对低地址分支直接把低 32 位作为函数地址调用。由于 SMEP 未开启，CPU 会以内核权限执行映射在用户空间的 `privesc`。

`privesc` 进入时从 `rdx` 保存一个已知内核地址，减去其无随机化偏移即可求 KASLR 基址。再修正 `prepare_kernel_cred` 和 `commit_creds` 地址，执行：

```c
commit_creds(prepare_kernel_cred(NULL));
```

最后用事先保存的 `CS`、`SS`、`RSP`、`RFLAGS` 构造 `iretq` frame，经 `swapgs; iretq` 返回用户态 `shell()`。UID 为 0 后读取：

```text
DUCTF{pac_must_stand_for_pwn_and_crypto_9b8a1bc12f2a39cc}
```

## 方法总结

本题虽包含密码分析，最终目标是伪造内核函数指针并完成 ring 0 提权，因此归入 Pwn。自制 PAC 同时犯了三类错误：tag 只有 32 位、已知明密文直接公开、S-box 又是有限域上的仿射映射；恢复密钥后，没有 SMEP 使合法签名的用户态指针可直接变成内核控制流劫持。
