# DEScent

## 题目简述

服务端主机制只有 3 个选项：

- `get_secret`：返回 `encoded_secret + error` 和随机 `user_seed`；
- `encode`：给定 `data,user_seed` 返回 `encode(data)+error`，且 `user_seed` 不能重复；
- `verify`：提交 `user_secret`（原始 16 字节）拿 flag。

`encode` 定义为把 byte 多项式求根（取 `ComplexField(128)` 根），`error` 来自 `DES.new(user_seed+server_seed).encrypt(randomness)`。
`DES` 密钥只用 8 字节（4 字节用户种子 + 4 字节服务种子），`encode` 中存在复杂数编码与随机项叠加。

官方说明强调了“同源 error 复用”，这正是题目的主漏洞面。

## 解题过程

源码 `admin/challenge` 中关键到可复现的是返回模型：

$$
\text{E\_s} = \text{encode}(SECRET) + \text{error}(user\_seed,server\_seed)
$$

`encode` 本身不是线性同态；攻击端只要拿到一个可控 `encode(data)`，并复现同一 `user_seed` 对应的同一 `error`，即可消去随机项：

$$
\text{encode}(SECRET)=\text{E\_s}-\text{error}
$$

官方 `admin/exploit/solve.sage` 做法：

1. 发送 `get_secret`，保存 `A`（`encoded_secret`）和 `user_seed`；
2. 将返回的 4 字节 `user_seed` 最后一个字节最低位翻转，再以这个新 seed 编码已知明文 `yellow submarine`。DES 每个密钥字节的最低位是奇偶校验位，不参与有效密钥调度，因此两个不同 seed 产生相同 error，同时绕过 `seen_seeds`；
3. `error = b - c`，其中 `b` 为可控消息编码；
4. `f = A - error`，把 `f` 转回 16 字节 secret；
5. `verify` 上提交恢复出的 `user_secret`。

仓库 `solve.sage` 已计算 `new_seed = old_seed XOR 00000001`，但随后请求中误用了旧 seed；照原样运行会命中 `Seed already used`。复现时应发送 `new_seed.hex()`，这是官方脚本与服务端当前版本之间必须修正的一行，而不是隐藏的额外攻击条件。

解码核心使用 `LLL`：

```sage
real = [(r ^ i)[0] for i in range(16)]
imag = [(r ^ i)[1] for i in range(16)]
M = matrix([[round(K * x) for x in real], [round(K * x) for x in imag]]).T.augment(
    matrix.identity(16)
)
data = bytes([abs(x) for x in M.LLL()[0][2:]])
```

注意 `K=2^(128-1)` 用于把复数根近似映射回字节格点。

## 方法总结

- 这题不是传统 DES 破解，主链路是“DES 相关 key 的重复错误项 + 复杂编码可逆化”。
- 复盘顺序：
  - 先拿 `get_secret`；
  - 再拿与其同错误项对应的 `encode`；
  - 用 `decode` 的 LLL 复原 `SECRET`；
  - `verify` 验证。
- 关键条件是 3 次查询窗口内完成：两次编码/查询 + 一次 verify。
- 最终服务返回 `bi0sctf{th4t_w4snt_t0o_c0mpl3x}`；本次未启动 Sage 服务，结论来自服务端、官方解题脚本说明与 README 的交叉核对。
