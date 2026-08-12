# 密码生成器

## 题目简述

网站透露管理员账号、注册时间（2021-09-22 23:11，UTC+8）以及密码由一个 Windows Qt 程序生成。生成器经过 UPX 压缩并静态链接，内部使用当前 Unix 秒作为 `std::mt19937` 的种子；所以密码空间并非 16 位复杂字符集的笛卡尔积，而只是注册时间附近的几十到一百多个 seed。

这一设计复现了卡巴斯基密码管理器漏洞的关键条件：用秒级时间初始化可预测 PRNG，攻击者知道密码生成时间后即可枚举候选；原实现还把两个均匀浮点数相乘来选字符，使低下标字符概率更高。

## 解题过程

### 去壳并定位业务函数

用 Detect It Easy 可识别 UPX，随后解压：

```powershell
upx.exe -d generator.exe -o generator-unpacked.exe
```

静态 Qt 代码量很大，直接浏览函数列表效率很低。搜索 UI 中的 UTF-8 字符串“错误: 未初始化 status”，交叉引用可定位到密码生成函数。IDA 若提示 `sp-analysis failed`，原因是函数序言动态执行 `sub rsp, rax`；在对应指令上手工标注栈变化 `-0x13e0`，再重新分析即可得到可读伪代码。

从初始化函数可以恢复状态结构：

```c
struct status {
    QString upper;
    QString lower;
    QString numeric;
    QString special;
    int length;
    bool use_upper;
    bool use_lower;
    bool use_numeric;
    bool use_special;
};
```

生成函数先拼接启用的字符表，然后执行：

```c
seed = time64(NULL);
mt19937_init(seed);
```

常量 `1812433253` 和长度 624 的状态数组是 MT19937 初始化的强识别信号。

### 动态枚举时间 seed

无需完全重写 `std::uniform_real_distribution<float>` 的实现，可以让原程序自己生成候选。以注册时间前后一分钟为范围，在 `_time64` 返回后覆盖 `RAX`，运行到密码生成完成处并读取 UTF-16 字符串：

```python
from ida_dbg import run_to, wait_for_next_event, WFNE_SUSP
from idaapi import get_bytes

seed = 1632323400  # 2021-09-22 23:10:00 +08:00
end = 1632323520   # 2021-09-22 23:12:00 +08:00

run_to(0xBC9393)   # time64 返回后
wait_for_next_event(WFNE_SUSP, -1)

with open("passwords.txt", "w", encoding="utf-8") as out:
    while seed < end:
        cpu.rax = seed
        run_to(0xBC9555)  # 16 字符密码生成完成
        wait_for_next_event(WFNE_SUSP, -1)
        password = get_bytes(cpu.rcx + 24, 32).decode("utf-16le")
        out.write(password + "\n")

        cpu.rip = 0xBC9518
        run_to(0xBC9400)
        wait_for_next_event(WFNE_SUSP, -1)
        cpu.rip = 0xBC9393
        seed += 1
```

这些地址对应题目给定二进制；换版本后应根据 `time64` 调用点、密码检查循环和 `QString` 布局重新定位。

### 在线验证候选

保留同一 Web 会话和 CSRF token，依次用候选登录 `admin`。成功密码为：

```text
$Z=CBDL7TjHu~mEX
```

登录后即可读取 flag。

[漏洞原型分析](https://donjon.ledger.com/kaspersky-password-manager/)还指出，类似 `int(r1 * r2 * alphabet.length())` 的索引生成会偏向较小下标；不过本题真正把搜索空间降到可枚举规模的是秒级 seed，而非分布偏差。

## 方法总结

- 核心技巧：去壳后用字符串交叉引用定位 Qt 业务函数，识别 MT19937 与 `time()` 种子，再借原程序动态枚举注册时间附近的密码。
- 识别信号：密码生成时间可知、PRNG 状态常量 1812433253/624、`time64` 直接作为 seed。
- 复用要点：预测 PRNG 时要复现具体标准库的浮点分布和字符选择语义；若跨语言重写困难，补丁或调试原二进制通常更可靠。
