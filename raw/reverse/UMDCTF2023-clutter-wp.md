# Clutter

## 题目简述

附件 `clutter.vsp` 是 VeSP（Very Simple Processor）机器码。该模拟器没有打印指令，程序只能修改寄存器和内存。题面说作者忘记把 flag 写到了哪里，实际程序会按执行顺序计算每个字符，却把结果写到
`0x0100` 至 `0x02ff` 的随机地址，制造“内存很乱”的假象。

随附的一页手册给出的必要运行信息是：使用能从文件读取程序的 VeSP 1.1，从起始地址 `002` 加载机器码，并打开 verbose 模式以观察指令和内存写入。

## 解题过程

把程序交给 VeSP 1.1 模拟器，选择：

```text
0   进入程序加载
002 起始地址
1   从文件读取
<clutter.vsp 的路径>
0   连续执行
1   verbose 输出
```

反汇编可以看到每个字符重复使用同一模式：

```text
LDA B, offset
LDA A, encoded_value
ADD
MOV random_address, A
```

生成器保证：

$$
\text{encoded\_value}+\text{offset}=\text{flag byte},
$$

然后把结果写入随机地址。地址本身不编码顺序；真正的字符顺序是这些 `MOV` 在执行轨迹中出现的顺序。

保存 verbose 日志，并只提取目标内存范围内的写入值：

```bash
./vesp1_1 | tee trace.log

grep -i "memory\\[" trace.log \
  | grep -E '0(1|2)[0-9A-F]{2}\\] = [0-9A-F]{4}' \
  | awk '{print $3}' \
  | cut -c 3- \
  | tr -d '\n' \
  | xxd -r -p
```

模拟器以 16 位字显示内存值，而字符位于低 8 位，所以 `cut -c 3-` 保留末两位十六进制。按轨迹顺序拼接后得到：

```text
UMDCTF{Ux13-us3-m3m0ry-w1p3!}
```

## 方法总结

- 核心技巧：理解 VeSP 的加载、加法和内存写入语义，从 verbose 轨迹按执行顺序提取低字节，而不是按随机内存地址排序。
- 识别信号：程序无输出指令，却频繁把小整数写入一段特定内存范围时，模拟器 trace 往往就是观察通道。
- 复用要点：区分“执行顺序”和“地址顺序”；随机地址只是干扰，日志中的每次写入值才连续组成明文。
