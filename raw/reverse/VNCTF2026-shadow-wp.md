# Shadow

## 题目简述

题目由用户态迷宫程序 `Maze.exe` 和内核驱动组成，参考 WP 来自 [VNCTF2026 Shadow WP](https://tkazer.github.io/2026/02/02/VNCTF2026-Shadow-WP/)。用户态程序本身只是简单迷宫，走到终点后会弹出 `You reached the end!`，并调用一次 `Sleep(0x32)`。真正的校验逻辑隐藏在驱动侧。

驱动第一层像壳：入口处用全局 key 处理一段约 `0x5E00` 字节的数据，得到 PE 头为 `MZ` 的内嵌驱动，再手工完成 PE 拉伸、重定位、IAT 修复并调用被加载驱动的 `DriverEntry`。被加载驱动才是核心逻辑。

核心驱动做了三件事：

1. 通过 PTE hook 只在 `Maze` 进程中 hook `KeDelayExecutionThread`。
2. 创建键盘设备并附加到 `\Device\KeyboardClass0`，在 `IRP_MJ_READ` 的完成例程中记录按键；按 F12 开始输入，再按 F12 结束输入。
3. 在 hook 函数中根据 `KeDelayExecutionThread` 第三个参数派生密钥，动态解密 shellcode；shellcode 加密记录到的输入并与密文比较，正确时触发 `0x11111111` 蓝屏作为成功信号。

关键细节是：`Sleep(0x32)` 的 50ms 传到底层 `KeDelayExecutionThread` 时，不是原始毫秒值，而是以 100ns 为单位的负数，即 `-500000`，对应 `0xFFFFFFFFFFF85EE0`。用该值派生出的关键密钥为 `0xE7D1CC85D2172C16`。

## 解题过程

先分析 `Maze.exe`，确认它只负责迷宫逻辑和终点触发，不直接校验 flag。终点处的 `Sleep(0x32)` 是和驱动 hook 交互的关键输入，因为驱动 hook 的就是 `KeDelayExecutionThread`。

驱动入口处先处理内嵌 PE。判断依据是处理后的数据出现 `MZ` 文件头，并且后续逻辑手动修复重定位和导入表，再跳转到另一个 `DriverEntry`。因此表层驱动主要是反射加载器，真正要分析的是内存中被加载的第二层驱动。

第二层驱动先解密字符串 `KeDelayExecutionThread`，用 `MmGetSystemRoutineAddress` 找到内核函数地址，再遍历进程定位进程名为 `Maze` 的目标进程，对该进程做 PTE hook。PTE hook 的效果是：同一个系统函数只在目标进程上下文中被替换，不影响全局系统行为。

同时驱动注册键盘读取回调，附加到 `KeyboardClass0`。完成例程会处理 Shift 状态和按键按下状态，把键盘消息转换为字符并存入全局 `Input` 数组。输入流程由 F12 控制：第一次 F12 开启记录，输入 flag，第二次 F12 结束记录。

之后分析 hook 函数。它利用 `KeDelayExecutionThread` 的第三个参数派生密钥，并分段解密一段 shellcode。由于用户态 `Sleep(0x32)` 对应的是 50ms，而内核参数单位是 100ns 的负数，所以需要转换：

```text
50 ms = 50,000,000 ns = 500,000 * 100 ns
Interval = -500000 = 0xFFFFFFFFFFF85EE0
```

用这个 interval 派生出 `0xE7D1CC85D2172C16` 后，可以解密 shellcode。shellcode 的算法不是标准密码算法，代码量不大，做法是把解密后的 shellcode 导入 IDA，按块逆出自定义 S-box、轮密钥和块解密逻辑。

静态解密时还会遇到一个陷阱：密文数组在验证前会被修改，但对密文本身找交叉引用不容易发现。原因是修改代码从密文前约 40 字节的位置开始，以越界方式影响到密文区域。更稳的办法是在 WinDbg 中用 `sxe ld Shadow` 断驱动加载，在反射加载后的 `DriverEntry` 处下断，再断到 `RtlCompareMemory` 的密文比较处；按 F12 输入后读取运行时真实密文，再用还原出的 shellcode 算法逆解。

## 方法总结

驱动逆向题要先区分用户态诱饵和内核态真实校验。这里 `Maze.exe` 的迷宫只是触发 `Sleep` 和提供进程上下文，真正逻辑在驱动的反射加载、PTE hook、键盘监控和动态 shellcode 中。

遇到 hook `KeDelayExecutionThread` 的题，要注意 `Sleep(ms)` 到内核 `Interval` 的单位和符号转换；直接拿毫秒值派生密钥会错。动态 shellcode 和运行时改密文这类逻辑，不要只依赖静态交叉引用，必要时在驱动加载点和最终比较点之间动态取证。
