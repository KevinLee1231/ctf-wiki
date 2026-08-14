# bi0sCTF 2025 - gettingWiser 题解

## 题目简述

这是一道 Windows 用户态程序与内核驱动协作的逆向题。用户态程序完成环境检查、解密并加载 `C:\\bi0s.sys`，随后通过 `DeviceIoControl` 把输入交给驱动。真正的校验被故意放在异常处理路径中：只有传入 65 字节数据，驱动才会因除零进入 `__except` 并执行完整检查。

仓库没有单独的官方 solve 脚本，但根目录 `README.md` 已给出官方短题解和 flag：

```text
bi0sctf{aBAHHCEUVDTINFuk7567r87kkjd3rtyyjsdogaeurrandom_stuffwerjgeorhdfy}
```

## 解题过程

用户态代码用 `input.length() + 1` 作为 IOCTL 输入长度：

```cpp
BOOL success = DeviceIoControl(
    hDevice,
    IOCTL_SEND_STRING,
    (LPVOID)input.c_str(),
    (DWORD)(input.length() + 1),
    NULL, 0,
    &bytesReturned,
    NULL
);
```

驱动的正常路径包含下面这行：

```cpp
data[0] = (char)(data[0] / (len - 66));
```

因此用户输入长度为 65 时，传给驱动的 `len` 为 66，除数变成 0，控制流进入结构化异常处理器。随后执行的真实判定为：

```cpp
bool correct = (*data == 'a')
    && checkBlowfishKey(data + 1)
    && checkInputPart1(data + 1,  data + 33)
    && checkInputPart2(data + 9,  data + 41)
    && checkInputPart3(data + 17, data + 49)
    && checkInputPart4(data + 25, data + 57);
```

这把 65 字节输入拆成三部分：

- `data[0]` 必须是字符 `a`；
- `data[1..32]` 是 32 字节 key，由 `checkBlowfishKey()` 内的栈式 VM 变换后，与 `enc_vm_key` 的 32 个字节比较；
- `data[33..64]` 是四个 8 字节块。每一块分别使用 key 的对应 8 字节作为 Blowfish 密钥，加密后与源码中的两个 32 位常量比较。

四组 Blowfish 目标值分别是：

```text
part 1: D6E782BB 665F6447
part 2: CE51F72E 281218CD
part 3: A39E5F8C 5F781C78
part 4: AD4DD79B 78E0AE72
```

每个 `checkInputPartN()` 的 `try` 块还会用初始化为 0 的 `fake[8]` 作除数，确保执行落入该函数自己的 `__except`；Blowfish 比较就藏在这个异常分支中。完整逆向顺序应当是：

1. 按 `vm_function()` 的逆序栈执行语义还原 32 字节输入 key，而不能直接把 `enc_vm_key` 当成明文 key；它是 VM 变换后的目标。
2. 以四段 key 初始化源码自带的 Blowfish，实现或调用解密过程，分别逆出四个 8 字节输入块。
3. 按 `a || key || block1 || block2 || block3 || block4` 拼成 65 字节，再经用户态程序发送。

这里必须指出证据边界：当前仓库只提供源码、成品程序和官方短题解，没有保存官方输入恢复脚本或 65 字节验签输入。仅凭 `enc_vm_key` 直接声称已经恢复 key 会产生错误结果；把该数组自身送入 VM 后，32 个输出字节全部不匹配。因此本文保留可由源码逐行确认的完整校验机制和官方 flag，不伪造缺失的中间输入。

## 方法总结

本题用两层异常控制流掩盖真实逻辑：外层以 `len == 66` 触发驱动异常，内层四次主动除零进入 Blowfish 比较。分析这类题时，应先固定缓冲区长度与偏移，再区分“变换前输入”和“变换后比较常量”。本题的官方 flag 来自仓库 `README.md`；驱动结构与所有比较常量来自当前 `driver.cpp`。由于缺少官方 solver，本文没有把未经验证的 VM 逆解冒充为已复现结果。
