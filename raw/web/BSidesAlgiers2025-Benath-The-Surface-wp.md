# BSidesAlgiers2025 - Benath The Surface

## 题目简述

本题是一个定制 HTTP 服务的控制题。服务有两处关键行为：

1. `/` 与静态文件读取接口允许把路径拼接后从本地文件系统取数，并支持 `Offset`/`Size` 头；
2. 服务端提供一个随机 CLI 路径，只有携带正确 RSA 签名的请求才能触发 `popen`。

官方材料里给出的 `src/chall.c`、`Dockerfile` 和 `solution/solve.py` 共同定义了完整机制。题面未给出任何“看起来像编码题”的线索，真正阻塞点是“如何在 HTTP 读取权限约束下拿到执行权限”。

## 解题过程

### 关键观察

在 `handle_request()` 内，除了 `/` 重定向和 CLI 分支外，静态分支会执行：

```c
char filename[PATH_MAX_SIZE + 0x10] = "./static";
strncat(filename, req->path, PATH_MAX_SIZE);
FILE *f = fopen(filename, "rb");
```

配合 `Offset`、`Size` 头处理（针对 `fseek` 与 `fread`）后，`/proc/self/*` 变成可利用的“内存读取接口”。

验证签名分支对每次请求会取 `c`、`s` 两个头，并用 `PUBLIC_KEY_PEM` 对 `c` 内容做 `SHA256withRSA` 验签。私钥不直接在文件系统；`PUBLIC_KEY_PEM` 在启动后从 `public_key.pem` 读入再 `remove`，随后放在堆内存，因此只要把关键内存读出来就能还原签名链路。

### 完整利用链

1. 先用路径穿越读取进程文件，验证服务模型并定位可访问范围：

```http
GET /../../../../../../../../proc/self/exe HTTP/1.1
```

2. 读取 `proc/self/maps`，提取一个稳定可用的 `bss_base`（官方脚本按 `mappings[4]` 解析）。
3. 用 `Offset: {bss_base}`、`Size: 1024` 读取 `proc/self/mem`，解析得到：
   - `pub_key_adr`（`data[0x40:0x48]`）
   - `endpoint`（`data[0x60:0x60+0x80+1]`）
4. 再读一次 `proc/self/mem` 到 `pub_key_adr` 区域，取出 `public_key.pem` 文本。
5. 解析 `(n,e)` 后，由于 `e` 异常大，按 Wiener's attack 恢复私钥 `(p,q,d)`。
6. 用恢复私钥签名命令，构造隐藏端点请求：

```python
from base64 import b64encode
from Crypto.Signature import pkcs1_15
from Crypto.Hash import SHA256
from Crypto.PublicKey import RSA

key = RSA.import_key(open("leaked/private.pem").read())

def construct_command_request(endpoint: bytes, cmd: str) -> str:
    signature = b64encode(
        pkcs1_15.new(key).sign(SHA256.new(cmd.encode()))
    ).decode()
    return f"""\
GET {endpoint.decode()} HTTP/1.1\r
c: {cmd}\r
s: {signature}\r
\r
"""
```

7. 发送 `ls -la / > /tmp/a.txt`，再读取该文件确认命令执行。
8. 最后读取题目 flag 文件（脚本提示输入）并输出。

仓库内 `challenge/flag.txt` 对应的最终结果为：

`shellmates{c4nT_be13iv3_h0w_d33p_y0u_w3nt!_y0uR_4_Tru3_M4$t3r}`

### 依赖脚本与重放流程

`solution/solve.py` 已完整给出上述链路，包含 `wiener` 恢复、签名请求拼装、LFI 回读 flag 文件的闭环。关键依赖是：

- `gmpy2`（`isqrt`）
- `pycryptodome`（`Crypto.PublicKey`、`SHA256`、`pkcs1_15`）
- `cryptography`（PEM 解析）
- `pwntools`（socket 交互）

### 验证

脚本在每个分步都以服务返回体是否成功为判断标准。若最终命令执行成功，`/tmp/a.txt` 能读回目录列表；随后输入 flag 文件名可触发最终 flag 输出。这个判断标准与源码机制一致，形成可复现闭环。

## 方法总结

- 关键技巧：将 path traversal 与 `Offset`/`Size` 结合，先把 LFI 扩展为进程内存读；把签名执行接口的关键材料（公钥与路径）从内存中恢复后再构造合法签名。
- 识别信号：出现可读 `/proc/self/exe`、`/proc/self/maps`、`/proc/self/mem` 且命令接口依赖签名时，优先判断为“读取内存找控制参数再签名调用”。
- 复用要点：签名校验不是边界本身；边界在于能否借由文件读取拿到可用于签名的上下文。服务端把 `command_path` 与公钥材料放在可导出的进程空间时，常见于此类“LFI + 反射签名”链路。
