# DownUnderCTF 2022 EVM Vault Mechanism Writeup

## 题目简述

目标合约直接用 Yul 编写。calldata 前 4 字节选择 `A` 到 `F` 中的一道锁，后 4 字节作为输入；通过某道锁后，合约把 storage slot `0x1337` 与对应常量异或。调用 `vrfy` 时，只有该槽等于 `0xff` 才会把 `storage[0x736f6c766564]` 置为 1。

不必攻破全部六道锁。只要选择一组可解分支，使其异或常量最终组合成 `0xff` 即可。官方解法选择 A、C、D、E：

$$
74\oplus100\oplus178\oplus99=255.
$$

## 解题过程

### A：逆运算

A 的变换只包含加法、乘法和异或，按相反顺序求逆：

```python
inp_a = ((0x346D81803D471 ^ 0xB3ABDCEF1F1) // 0x80) \
        - 0x69B135A06C3
send(target, b"AAAA", inp_a.to_bytes(4, "big"))
```

### C：构造满足环境约束的调用者

C 同时检查调用者地址最低字节为 `0x77`、余额大于 1 ether、代码长度等于代码哈希中的一个派生字节，以及调用者 runtime code 指定 4 字节片段的哈希最低字节为 `0x77`。官方 solver 预先构造满足代码长度和片段哈希的短 runtime bytecode，再根据 `CREATE` 地址由部署者和 nonce 决定的性质推进 nonce，直到下一合约地址以 `0x77` 结尾。部署时附带 1.01 ether，随后由该攻击合约调用目标的 `CCCC` 分支。

### D：让旧区块哈希为零

D 读取：

```yul
blockhash(number() - (3 + ((2 * low_byte) & 0xff)))
```

输入 `0x1a001a7f` 令回溯距离为 257。EVM 对超过最近 256 个区块范围的 `blockhash` 返回 0；同时输入的其它字段让表达式两侧都为 `0x1a1a`，因此校验成立。

### E：代码哈希字节子集和

E 读取合约自身 `extcodehash` 的 32 个字节，并用输入的 32 位掩码选择其中 17 个字节，要求其和模 1337 等于 777。目标 bytecode 固定，因此先读取代码哈希，再求一个满足计数和模约束的子集即可；官方实例使用：

```python
inp_e = 0xB23B606F
send(target, b"EEEE", inp_e.to_bytes(4, "big"))
```

完成四次调用后检查 `storage[0x1337] == 0xff`，再发送 `vrfy`。平台确认 solved slot 后返回：

```text
DUCTF{b3y0nd_th3_v4ult_li3s_a_w3ll_d3serv3d_fl4g}
```

## 方法总结

这道题考查 EVM 边界语义而非单一漏洞：模运算的可逆性、`CREATE` 地址可预测、`EXTCODE*` 环境约束、`blockhash` 仅保留最近 256 块，以及固定代码哈希上的子集和。面对多分支状态机时，先把每个分支对最终状态的贡献写成代数关系，再挑选总效果满足目标且求解成本最低的子集，通常比逐一破解所有分支更直接。
