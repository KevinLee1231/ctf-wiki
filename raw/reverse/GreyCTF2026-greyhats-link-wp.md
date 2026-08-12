# Greyhats Link

## 题目简述

目标是一个只剩固件恢复页面的路由器。公开包包含定制 Lua 5.4 解释器和编译后的 CGI bytecode；上传镜像必须通过自定义 4×4 矩阵“签名”校验，成功后 CODE section 会覆盖当前 `index.cgi`。需要先恢复被轮换的 Lua opcode 语义，再还原固件格式与校验算法，构造能够读取 `/flag.txt` 的合法固件。

HTTP 只负责传输上传文件，决定性障碍是定制解释器和固件验证逻辑的逆向，因此归入 Reverse。

## 解题过程

用标准 Lua 反编译器处理 `index.cgi` 时，加法、乘法和按位 XOR 会出现在不合理的位置。对定制解释器与标准 Lua 5.4 opcode 表进行比对，可恢复三向轮换：

```text
真实 XOR -> 标准 ADD 槽位，普通反编译显示为 +
真实 MUL -> 标准 BXOR 槽位，普通反编译显示为 ~
真实 ADD -> 标准 MUL 槽位，普通反编译显示为 *
```

`ADDK/MULK/BXORK` 和相应 metamethod/运算符枚举也做了同样轮换。按上述映射修正反编译结果后，CGI 的固件解析逻辑变得清晰。

镜像至少 0x38 字节，以 `GREY` 开头。主要字段为：

| 偏移 | 字段 |
| ---: | --- |
| `0x08` | header size，小端 `u32` |
| `0x0c` | section 数量，小端 `u32`，最大 64 |
| `0x34` | payload size，小端 `u32` |
| `0x38` | section 表，每项 16 字节 |

每个 section entry 依次是 `type, offset, size, flags`。类型 `0x01` 是最终执行的 CODE，类型 `0x06` 是 16 字节 CERT。section 数据都位于 payload 中，偏移相对 payload 起点。

校验先按表顺序连接所有非 CERT 数据。若长度不是 16 的倍数，则用 `pad_len` 重复填充到整块。状态从 $4\times4$ 单位阵开始；第 $i$ 个块与 LCG 生成的轮常量逐字节 XOR，按行优先解释成 $B_i$，然后在 $\mathbb Z/251$ 上更新：

$$
H_0=I_4,\qquad H_{i+1}=H_iB_i\pmod{251}
$$

CERT 的 16 字节同样按行优先解释为矩阵 $S$，验证条件只是：

$$
HS=I_4\pmod{251}
$$

这不是带私钥的签名。只要为自己选择的 CODE 计算 $H$，并确保 $\det(H)\ne0$，就能直接取：

$$
S=H^{-1}\pmod{251}
$$

构造的 Lua payload 为：

```lua
io.write("Content-Type: text/html\r\n\r\n")
local f = io.open("/flag.txt", "r")
local flag = f and f:read("*a") or "flag unavailable"
if f then f:close() end
io.write(flag .. "\n")
```

生成两个 section：先放 CODE，再放 `mat_to_bytes(H^-1)` 作为 CERT；设置 header size 为 `0x38 + 2 * 16`，并填入各自相对偏移和长度。上传时使用 multipart 字段名 `firmware`。

CGI 校验成功后执行：

```lua
local out = io.open(arg[0], "wb")
out:write(code)
```

即用 CODE 覆盖当前 CGI 文件。第一次 POST 只会显示升级成功，再请求一次 `/cgi-bin/index.cgi`，定制 Lua 解释器就会执行刚写入的源码并打印当前实例 flag。仓库中的静态模板为：

```text
grey{i_love_luanear_algebra_UUID}
```

正式 Whale 实例会把 `UUID` 替换为每容器生成的后缀，应以实际响应为准。

## 方法总结

本题把 opcode 混淆、固件容器和代数校验串在一起。先修正三种运算的语义，才能识别所谓 hash 是有限域矩阵连乘；一旦看到验证式 $HS=I$，就能发现“签名”没有任何秘密，只是公开 hash 的逆矩阵。最后利用升级逻辑覆盖 CGI，下一次请求即执行任意 Lua。每一步都可用 $H\cdot H^{-1}=I$ 和服务的成功页做明确验证。
