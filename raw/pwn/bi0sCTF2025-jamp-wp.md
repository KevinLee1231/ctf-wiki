# JAMP

## 题目简述

题目是 “Just Another MIDI Parser” 文件解析题：程序要求传入二进制文件后执行 `parseMIDI()` 并打印 track/event 信息。交互逻辑极简：`./midiocre <File>`，因此核心是输入文件格式可控性而非运行时协议。

官方题面和短解都指向两个要点：`Parser has an OOB increment bug` 与 `leakless` 场景。真正被官方 exploit 使用的原语出现在音符统计表：表长只有 $0x7f$，MIDI Note-On 事件的 `parameter1` 却是任意 8 位值，写入前也没有边界检查。攻击者因此能对统计表之后的堆内存做逐字节递增。

## 解题过程

### 关键观察

`resetOctaves()` 创建两个长度为 $0x7f$ 的向量，合法下标只能是 $0$ 到 $0x7e$：

```cpp
Octves->notes.note = std::vector<char>(OCTAVE_FF, 0);
Octves->notes.count = std::vector<uint8_t>(OCTAVE_FF, 0);
```

解析 Note-On 事件时，`parameter1` 直接来自文件中的一个字节。随后 `setOctaveTbl()` 把它当作向量下标，并对对应计数器执行递增：

```cpp
void MIDI::setOctaveTbl(MIDI_EVENT *event) {
    (*(Octves->notes.count.begin() + event->parameter1))++;
    Octves->noteBuf.push_back(
        *(Octves->notes.note.begin() + event->parameter1)
    );
}
```

当 `parameter1 >= 0x7f` 时，第一句形成越界的单字节加一，第二句还会发生越界读。源码的 running-status 分支另有 `rbegin() + 1` 这种脆弱访问，但官方 `incbyte()` 利用链针对的是上述无边界音符索引，不能把两处缺陷混为一谈。

### 利用链

1. 构造 MIDI 头与 track 结构

利用 `Mheader` / `MTrack` / `TMETA` / `TMIDI` 封装函数，先布置可控布局。

```python
def Mheader(leng,fmt,ntracks,div):
    return b"MThd" + b32(leng) + b16(fmt) + b16(ntracks) + b16(div)

def MTrack(leng):
    return b"MTrk" + b32(leng)
```

2. 用 `incbyte`/`transfer` 批量生成 Note-On 事件流

```python
def incbyte(idx, amt):
    return TMIDI(0x0,0x90 | MTYPE_NOTE_ON,idx,0x0)*amt

def transfer(nbytes,idx):
    pll = b""
    for i in range (nbytes):
        pll += incbyte(idx + i,1)
    return pll
```

每个 Note-On 事件都会让 `count.begin() + idx` 指向的字节增加一次；重复 `amt` 次即可把目标低字节调整到需要的值。`transfer()` 则依次触碰相邻下标，帮助搭建连续字段。

3. 堆风水后将递增原语落到 `FILE` 相关对象

官方脚本先用多组 Lyrics 元事件和多个 track 分配不同尺寸的向量，填充空洞并稳定统计表、流对象与相关缓冲区的相对位置。之后通过多段 `incbyte()`：

- 调整已有 libc 指针的低字节，使目标调用槽落到 `system`；
- 改写 `FILE` 对象及其 `_wide_data` / wide-vtable 指针；
- 写入 `p;sh` 作为可被 shell 解释的文件名/命令片段；
- 让程序退出时的关闭路径使用伪造表。

这是 leakless 利用：脚本依赖题目配套 libc 与受控堆布局，以相对字节调整完成指针变换，而不是先从服务输出中取得地址。

4. 触发输出/析构链

作者注释给出“设置 vtable 中的 `system` 指针 -> 覆盖 `wide_data->vtable` -> 进入与 `fclose` 有关的 FSOP 路径 -> 执行命令”。因此最终触发点不是解析器立即跳转，而是解析结束后的流关闭/析构。

5. 最终执行

```bash
./JAMP ./examples/exploit.mid
```

### 验证

官方 exploit 是端到端 PoC：先生成 `examples/exploit.mid`，再以该文件启动本地程序；若布局与配套 libc 一致，解析结束时会触发被篡改的关闭链并执行命令。当前没有实际运行 PoC，因此这里只把官方脚本与源码能够证明的利用路径作为验证依据，不把“本机已拿到 shell”写成事实。

## 方法总结

- 核心技巧：用未经范围检查的 MIDI 音符编号越界索引 `std::vector<uint8_t>`，获得堆上的单字节递增，再通过 FSOP 劫持关闭路径。
- 识别信号：固定长度统计表接收更宽的文件字段，且操作是 `table[index]++` 时，应同时检查越界范围、可重复次数和相邻堆对象。
- 复用要点：单字节增量不等于任意写；需要先用堆风水稳定目标，再利用已知指针低字节和重复递增完成定向改写。leakless 方案还强依赖配套 libc 与布局，复现时必须保持环境一致。
