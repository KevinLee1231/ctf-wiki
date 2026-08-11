# good game well played

## 题目简述

这是一个双参与方 CMP-ECDSA/MtA 协议服务。服务先完成密钥建立，再不断与客户端交换 JSON 序列化的 commitment、Paillier 证明、MtA request/response 和 delta；客户端最后可提交 32 字节私钥。若其与服务端保存的 `downunderctf` 私钥完全相同，服务输出 flag。

题目源码 `challenge.diff` 删除了 Paillier Blum 证明验证中对 `proof.w` 与模数互素的检查。手册说明构建时会拉取 MPC 库并应用这个补丁。这个单点“省略检查”使攻击者可让自选、已知因数的 Paillier 密钥通过建立阶段，并把后续 MtA 中本应受范围证明保护的计算变成服务端私钥的模小素数泄露。

## 解题过程

### 接受攻击者可解密的 Paillier 参数

官方 `solve/solve.diff` 是为攻击客户端编译 MPC 库的补丁，不是远程服务补丁。它将本地生成的 Paillier $n,p,q,\lambda,\mu$ 固定为一组已知值，并令 Paillier Blum 证明中的 `w` 取 $n$。正常验证器会因 $\gcd(w,n)\ne1$ 拒绝；题目删除该检查后，服务接受这个客户端的建立数据。于是攻击者能够解密发送到自己 Paillier 公钥的 MtA 响应。

这是本题必须保留的实现边界：普通 CMP 客户端不会输出后面的中间明文；官方 solve 通过修改本地库把解密后的 `alpha` 复制到调用者可读的 `BIGNUM`，并在读取后提前结束本轮签名。

### 对每个小素数取一条私钥同余

设服务端待恢复的标量为 $x$，攻击者依次选择约 16 bit 的互异素数 $p_i$，并在其修改后的客户端中令 MtA 相关输入为

$$
k=\frac{n}{p_i}.
$$

攻击客户端还会反复生成 range proof，直到其中 Fiat--Shamir challenge 满足 $e\equiv0\pmod {p_i}$。`solve.diff` 中的 `BN_mod(tmp, e, p, ctx)` 与循环条件正是这一筛选。其余被官方 solver 置零或改写的中间标量、以及服务端漏掉的 Paillier 证明检查，保证这一畸形 MtA 轮次仍能推进到服务端对 `k_x_mta` 的计算。

从该响应解出的整数记为 `alpha`，官方 solver 对每轮直接执行：

下面是与官方提交等价的伪代码；实际 `guess` 的 Base64 编码由 handout 的 `serialization.hpp` 完成。

```cpp
div = n / p_i;
sub = alpha % div;
residue = (alpha - sub) / div;
```

在这组恶意参数下，`residue` 就是服务端私钥的 $x\bmod p_i$。因而无需理解完整 ECDSA 签名结果，只要在 RPC/JSON 协商期间记录这一个 Paillier 明文即可。

### CRT 合并并提交

官方 solver 内置从 32783 开始的一组小素数。每轮成功后收集 $(r_i,p_i)$，当收集到 17 条时以中国剩余定理恢复

$$
x\equiv r_i\pmod {p_i}.
$$

这些素数乘积已超过 secp256k1 标量的可能范围，故 CRT 代表元唯一对应服务端的 256 bit 私钥。将该数按 32 字节大端编码为 JSON `guess`，并发送 `{"choice":"submit_key","guess":...}`；服务端 `memcmp` 成功后返回 flag。

```cpp
BIGNUM *key = crt(residues, primes, ctx);
std::vector<uint8_t> guess(32, 0);
BN_bn2binpad(key, guess.data(), guess.size());
send_json({{"choice", "submit_key"}, {"guess", base64(guess)}});
```

官方实现的实际序列化由 handout 中的 `serialization.hpp` 完成：字节数组会 Base64 编码，不能把上例中的 `base64` 换成十六进制字符串。

### 验证

官方 C++ solver 在每轮将解密出的 `alpha` 化为一个 residue，收集 17 条后运行 CRT 并持续提交 key。题目配置给出的验证材料为 `DUCTF{w0w_pailli3r_skip_sav3s_12_s3c0nds_0ff_th3_any_p3rc3nt_run}`。本归档未编译或运行这套定制 MPC 客户端；协议级结论仅来自题目 `challenge.diff` 与官方 `solve.cpp`/`solve.diff` 的静态对照。

## 方法总结

- 核心技巧：在阈值签名/MtA 协议中，Paillier 证明的合法性是防止恶意参与方选择退化参数的安全边界；漏掉互素性检查可将私钥降为一组可 CRT 合并的小模同余。
- 识别信号：源码把“证明中某个值必须与模数互素”当作可选优化删除，且攻击者能提供 Paillier 公钥、证明或 MtA request 时，应优先审计恶意 modulus/密文能否让解密值出现可控因子。
- 复用要点：先记录官方 solve 真正读取的中间量与公式，再决定应重建哪一段协议。本题的 JSON/Base64 编码、Fiat--Shamir 筛选和本地 MPC 库补丁缺一不可；不要把 CRT 本身误当成漏洞根因。
