# Umiyuri Kaikeitan

## 题目简述

题目是 Windows x64 程序。程序把 `.text` 按页异或加密并设为 `PAGE_NOACCESS`；执行到某页时，向量化异常处理器根据 `codeStatus[idx].key` 解密该页，再改为可执行。它还包含多组反调试检查。

`main()` 读取 `data.txt`，逐字节与 `PJSK` 循环异或后写入全局数组：

```cpp
char staticStorage[65536];
CodeStatus codeStatus[1024];

while (!feof(f)) {
    staticStorage[i++] = fgetc(f) ^ key[k];
    k = (k + 1) % 4;
}
```

循环没有边界检查，而且 `codeStatus` 紧跟在 `staticStorage` 后面。攻击者可以用超长文件覆盖代码页的解密密钥和 `encrypted` 状态，使程序的自解密过程变成自修改代码利用链。

## 解题过程

### 1. 输入前 0x10000 字节放 shellcode

官方生成器先生成 Windows `WinExec` shellcode，执行：

```text
cmd /C "start flag.png"
```

再用 `0xCC` 填充至 65536 字节：

```python
shellcode = asm(
    shellcraft.amd64.windows.winexec(
        b'cmd /C "start flag.png"'
    )
)
payload = shellcode.ljust(0x10000, b"\xCC")
```

因为程序读入时会与 `PJSK` 异或，最终写入 `data.txt` 前还要把整个 payload 再异或一次；程序解码后，内存中才恢复为上述字节。

### 2. 越界覆盖 `CodeStatus`

每个状态结构含两个 64 位 key 和一个 `BOOL encrypted`，在 x64 对齐后占 24 字节。越过 `staticStorage` 后，输入可精确定位到对应代码页状态。

正常解密逻辑对页内每 16 字节执行：

```cpp
base[off]     ^= codeStatus[idx].key[0];
base[off + 1] ^= codeStatus[idx].key[1];
```

若已知某处原始密文字节、正常 key 和希望得到的新指令，则替换 key 可按：

$$
K'=C\oplus P'\;=\;P\oplus K\oplus P'
$$

计算。官方脚本正是用：

```python
old_code ^ new_code ^ old_key
```

生成三个目标页的新 key，并把对应 `encrypted` 标志设为 1。

### 3. 让异常处理器替我们打补丁

官方补丁分三段：

1. 改写一处 `VirtualProtect` 参数，把输入区大小设为 `0x10008`、保护属性设为 `0x40`，即 `PAGE_EXECUTE_READWRITE`；
2. 改写另一页，把 `staticStorage` 地址送入后续调用链；
3. 改写最终调用点，从寄存器派生输入区地址并跳转进去。

这些页并非提前明文写入。程序正常运行并访问到相应页时，异常处理器使用被覆盖的新 key，“解密”出攻击者期望的新指令。于是自保护机制自己完成了 `VirtualProtect` 和控制流改写。

### 4. 执行输入区

第三段补丁最终跳到 `staticStorage` 开头，执行第一阶段放入的 shellcode，打开 `flag.png`。题目服务端会观察程序是否成功显示该图片，而不是要求传统交互 shell。

末尾 `while (!feof(f))` 还会多处理一次 EOF 返回值，但官方 payload 的布局和补丁偏移已把这一行为计入生成结果，复现时不应随意删改最后一个字节。

仓库正式挑战说明给出的图片内容为：

```text
SEKAI{t4p_sl1d3_t4p_sl1d3_r1ns3_4nd_r3p34t}
```

## 方法总结

这道题把数据溢出与自修改代码组合在一起。直接覆盖代码页不可行，因为页面不可访问；但解密元数据是可写全局数据，篡改 key 后，合法异常处理流程会把预期明文替换成攻击者代码。

面对 packer 或按页解密器，应同时审计：

- 密钥、页状态和函数指针是否与用户缓冲区相邻；
- 页索引是否有范围检查；
- “密文 XOR key”是否允许通过改 key 定向修改明文；
- 解密后权限是否过宽。

保护代码的元数据本身必须放在不可被业务输入越界触及的区域，并在使用前校验完整性。
