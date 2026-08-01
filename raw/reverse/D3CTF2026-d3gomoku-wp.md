# d3gomoku

## 题目简述

题目实现了一个运行在 Windows 内核驱动中的五子棋游戏。公开路径提供初始化、重置、落子、悔棋和查询等 IOCTL，负责维护棋盘、历史记录和 AI 落子；真正的胜利条件与 flag 解密逻辑则位于驱动运行时释放的第二阶段 PE 中。

第二阶段是 Intel VMX hypervisor。它通过 EPTP switching 为同一线性地址准备两套视图：静态读取时看到含诱饵常量的 clean view，执行时则切换到含真实常量的 hook view；同时通过 MSR bitmap 伪造证明计算所需的盐值。解题目标不是在普通规则下战胜 AI，而是构造十轮合法棋谱，使两条 base-37 证明轨道命中隐藏目标，并让坐标相关的 39 字节正文通过两层 SHA-256 校验。

这条动态路径依赖 Intel VMX、EPT 和 `VMFUNC`，应在支持 nested VT-x 的 Intel 环境中运行。最终 flag 为：

```text
d3ctf{Hey5h4d0ww4lk3R-1d2efadd-aaef-zenu-s100}
```

## 解题过程

### 1. 定位公开落子链与第二阶段

设备控制分发函数位于 `0x1400319C0`，公开 IOCTL 为：

| IOCTL | 作用 |
| --- | --- |
| `0x222040` | 初始化 |
| `0x222044` | 重置 |
| `0x222048` | 落子 |
| `0x22204C` | 悔棋 |
| `0x222050` | 查询 |

落子主分支从 `0x140036E1F` 开始，在 `0x1400372B0` 附近依次完成人类落子、AI 响应和隐藏门检查。第一阶段中可见的字符串 `ACELETMEIN` 位于 `0x1400312F0`，只是一处诱饵 marker；真正被 EPT 改写的入口是 `0x1400311C0`，隐藏比较最终进入第二阶段的 `0x1400184F0`。

第二阶段导出入口位于 `0x140018330`，停止入口位于 `0x1400184C0`。其中 `0x140010841` 是 EPT 视图切换后的桥接点：它先切回 clean view，再把执行视图中装入 `R10`、`R11` 的两个真实目标传给比较函数。

### 2. 从 EPT 双视图取得真实目标

clean view 中能静态看到的终值是：

```text
0x3CA5218358266D71
0x409B7C7DB5881627
```

只沿第一阶段或 clean view 分析会把这两个数误认为证明目标。hook view 中的补丁代码实际装入：

```asm
mov      r9d, r9d
movzx    r8d, r8b
movzx    edx, dl
mov      r10, 0x44EA257DE1CEFB27
mov      r11, 0xEED3C641A4C3A7A7
mov      rax, 0x140010841
jmp      rax
```

因此两条证明轨道的真实终值为：

```text
target_1 = 0x44EA257DE1CEFB27
target_2 = 0xEED3C641A4C3A7A7
```

这正是 EPT 双视图的作用：读取和反编译看到诱饵页，CPU 真正执行时却从另一物理页取得常量。

### 3. 逆推 base-37 证明

隐藏路径使用的静态盐值为 `0x00D31145`，第二阶段截获 `rdmsr` 后伪造返回 `0x00D10155`。两者异或得到最终盐值：

```text
salt = 0x00D31145 ^ 0x00D10155 = 0x00021010
```

设第 $i$ 轮人类坐标为 $(h^x_i,h^y_i)$，AI 坐标为 $(a^x_i,a^y_i)$，证明函数只使用两组坐标和：

$$
A_i=h^x_i+a^x_i,
\qquad
B_i=h^y_i+a^y_i.
$$

前一轮状态初值为：

$$
p_0=(salt\mathbin{\gg}1)\mathbin{\&}0xf=8,
\qquad
q_0=(salt\mathbin{\gg}5)\mathbin{\&}0xf=0.
$$

每轮先计算：

$$
\begin{aligned}
c_i&=(\operatorname{byte}(salt,i\mathbin{\&}3)+7i+19)\bmod37,\\
d_i&=(\operatorname{byte}(salt,(i+1)\mathbin{\&}3)+11i+29)\bmod37,\\
x_i&=(A_i+p_i+2B_i+c_i)\bmod37,\\
y_i&=(B_i+q_i+3A_i+d_i)\bmod37.
\end{aligned}
$$

再更新两条 64 位轨道以及前一轮坐标和：

$$
X\leftarrow37X+x_i,
\qquad
Y\leftarrow37Y+y_i,
\qquad
p_{i+1}=A_i,
\qquad
q_{i+1}=B_i.
$$

两条轨道的初始累加器由盐值经过 `fmix64` 得到：

```text
initial_X = 0x99998E967E16F091
initial_Y = 0x749434B6121A32BC
```

因为 $37^{10}<2^{64}$，十轮以内不会产生回绕歧义。对候选轮数 $n$，先计算：

$$
D=(target-initial\cdot37^n)\bmod2^{64}.
$$

只有 $D<37^n$ 时，该轮数才可能成立；把 $D$ 展开为恰好 $n$ 位 base-37 数字，就得到每轮的 $x_i$ 或 $y_i$。对每一轮令：

$$
u=(x_i-c_i-p_i)\bmod37,
\qquad
v=(y_i-d_i-q_i)\bmod37,
$$

则坐标和可直接解为：

$$
A_i=(2v-u)\cdot5^{-1}\bmod37,
\qquad
B_i=v-3A_i\bmod37.
$$

轮数从 1 到 9 均无解；在第 10 轮得到唯一合法的坐标和序列：

```text
(5,1), (3,8), (3,12), (5,12), (0,18),
(0,12), (9,3), (8,5), (6,7), (5,4)
```

### 4. 搜索唯一合法棋谱

证明只约束 $(A_i,B_i)$，每组坐标和仍可拆成多种人类与 AI 落子。必须按原始五子棋逻辑逐轮枚举人类落点，让原始 AI 走一步，再检查：

- 两枚棋子的坐标和是否等于本轮目标；
- 落点是否越界、重复；
- 棋盘状态是否仍然合法；
- 十轮结束后的正文校验是否通过。

唯一同时满足证明和正文校验的棋谱为：

| 轮次 | 人类落子 | AI 落子 | 坐标和 |
| ---: | ---: | ---: | ---: |
| 1 | `(2,0)` | `(3,1)` | `(5,1)` |
| 2 | `(0,6)` | `(3,2)` | `(3,8)` |
| 3 | `(0,9)` | `(3,3)` | `(3,12)` |
| 4 | `(2,8)` | `(3,4)` | `(5,12)` |
| 5 | `(0,8)` | `(0,10)` | `(0,18)` |
| 6 | `(0,7)` | `(0,5)` | `(0,12)` |
| 7 | `(4,1)` | `(5,2)` | `(9,3)` |
| 8 | `(4,3)` | `(4,2)` | `(8,5)` |
| 9 | `(4,5)` | `(2,2)` | `(6,7)` |
| 10 | `(1,0)` | `(4,4)` | `(5,4)` |

这 20 个落点互不冲突，双方在十轮内都不需要形成普通五连。隐藏检查根据证明状态判定人类胜利，而不是复用公开棋盘的普通胜负结果。

### 5. 恢复 39 字节正文并验证

第二阶段保存的 39 字节初始密文为：

```text
3b0fe8d33280a674c76a422d65ba6724d5fb2abfb8f5f7fd1a9ac555d1fea28e89de2ea7ccc7e4
```

每处理一轮历史，程序都会把人类和 AI 的四个具体坐标混入 keystream。因此，只满足坐标和而任意拆分落点并不能得到正确明文；这也解释了为什么搜索棋谱时还要保留正文校验。

正确十轮序列解出的正文十六进制为：

```text
48657935683464307777346c6b33522d31643265666164642d616165662d7a656e752d73313030
```

转为 ASCII 得到：

```text
Hey5h4d0ww4lk3R-1d2efadd-aaef-zenu-s100
```

固定校验密钥为 `0xB582A7EB`。第一层摘要验证正文：

$$
H_1=\operatorname{SHA256}(
\operatorname{LE32}(0xB0D7A925)\parallel
\operatorname{LE32}(0xB582A7EB)\parallel
\operatorname{LE32}(39)\parallel body).
$$

第二层摘要再绑定运行状态：

$$
\begin{aligned}
H_2=\operatorname{SHA256}(&
\operatorname{LE32}(0xD3E7A251)\parallel
\operatorname{LE32}(0xB582A7EB)\parallel
\operatorname{LE32}(counter)\parallel
\operatorname{LE32}(generation)\parallel\\
&\operatorname{LE32}(39)\parallel
\operatorname{LE32}(state_{60})\parallel
\operatorname{LE32}(state_{64})\parallel
\operatorname{LE32}(state_{68})\parallel H_1).
\end{aligned}
$$

两层摘要均通过后，第二阶段把正文包装为 `d3ctf{...}`，并将完成状态和完整 flag 写入：

```text
HKLM\SYSTEM\CurrentControlSet\Services\drv_game\Parameters
  RemoteFlagStatus
  RemoteFlagReady
  RemoteFlag
```

按顺序完成握手、触发第二阶段、重置游戏并走完上述十轮棋谱，即可从 `RemoteFlag` 读取：

```text
d3ctf{Hey5h4d0ww4lk3R-1d2efadd-aaef-zenu-s100}
```

## 方法总结

- 第一层陷阱是 EPT 双视图：clean view 中的两组常量是诱饵，只有跟进 `VMFUNC` 与 hook view 才能取得真实证明目标。
- 隐藏证明是两条 base-37 轨道。利用 $37^{10}<2^{64}$ 可以直接逆展开每轮数字，再解模 37 的二元一次方程得到唯一坐标和。
- 坐标和不是完整答案。逐轮正文 keystream 依赖人类与 AI 的四个具体坐标，必须用原始棋盘和 AI 逻辑搜索唯一合法拆分。
- 最终结果同时受 64 位证明、39 字节正文摘要和运行状态摘要约束；三者全部通过后，第二阶段才会包装 flag 并写入注册表。
