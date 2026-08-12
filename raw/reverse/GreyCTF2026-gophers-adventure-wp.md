# Gopher's Adventure!

## 题目简述

服务页面只加载 `main.wasm` 和 Go 的 `wasm_exec.js`。从 WebAssembly 还原出的逻辑保存了 4 个 32 位 checkpoint、16 字节加密 key、S-box 及其逆表，以及一段循环异或的 flag 密文。决定性工作是理解 WASM/Go 还原后的数据流，因此归为 reverse；S-box 只是一段静态可逆变换，不构成独立密码分析题。

每个 checkpoint 的四个字节依次参与 key 还原。随后 16 字节 key 循环异或 `enc_flag`，没有远端状态或暴力需求。

## 解题过程

### 还原循环 key

反编译代码给出：

```go
checkpts := []uint{0x00000100, 0x00005555, 0x01234567, 0x67676767}

func decryptKey(key uint, n int) {
    for i := range 4 {
        keys[i+n*4] = InverseSTable[enc_keys[i+n*4] ^ STable[uint8(key&0xff)]]
        key >>= 8
    }
}
```

因此第 $n$ 个 4-byte block 从 checkpoint 的低字节开始（小端顺序）恢复：

$$
K_{4n+i}=S^{-1}\bigl(E_{4n+i}\oplus S((C_n\mathbin{>>}8i)\mathbin{\&}255)\bigr),\quad 0\leq i<4.
$$

`E` 是 `enc_keys`，`S` 和 $S^{-1}$ 分别是源码中 256 项的 `STable`、`InverseSTable`。这一步也解释了为什么只逆查一张表即可：加密 key 的变换没有跨字节依赖。

### 解开 flag

还原 key 后，解密就是循环 XOR：

```go
for i := range len(enc_flag) {
    flag[i] = enc_flag[i] ^ keys[i%16]
}
fmt.Println(string(flag))
```

仓库提供的 `solve/main.go` 已是上述 WASM 逻辑的最小还原，可直接运行：

```bash
go run solve/main.go
```

输出为：

```text
grey{G0pHeR_g0e5_oN_4N_4dv3ntur3!XDDDD}
```

## 方法总结

- 核心技巧：从 WASM 还原数据表及循环结构，再按 byte-order 重建短 key。
- 识别信号：WASM 题若出现 256 项 S-box、对应 inverse 表、固定 checkpoint 与 `% key_length` 的 XOR 循环，通常是可直接静态逆向的自定义 key schedule。
- 复用要点：明确记录移位方向和低字节优先顺序；先用源码/反编译中的循环边界验证 16-byte key 周期，再输出文本，避免把 WASM 的整数表示误当成大端数据。
