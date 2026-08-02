# TSGCTF2024 Misbehave

## 题目简述

程序读取最多 47 字节输入，以 4 字节为一组调用自定义伪随机生成器并与输入异或，再与 48 字节常量 `flag_enc` 比较。表面流程为：

```c
for (int i = 0; i < LEN / 4; i++) {
    uint32_t r = gen_rand();
    *(uint32_t *)(buf + i * 4) ^= r;
    if (memcmp(buf + i * 4, flag_enc + i * 4, 4) != 0) {
        success = false;
    }
}
```

异常之处在于 `init(0x2cb7, 0x22)` 会自修改进程状态，实际执行的 `memcmp` 并不是 libc 的普通比较函数。仓库中的官方英文 WP 仍是 `TODO`，因此需要结合源码和官方 solver 还原隐藏逻辑。

## 解题过程

### 1. 找到 `init` 的 GOT 改写

程序以 `-z lazy` 编译，`init` 先取得当前 RIP，再按参数计算两个相对地址：

```c
state = 0xfeedf00ddeadbeef;
lea_current_rip(&rip);
*((uint64_t *)(rip + arg1)) = rip + arg2;
```

本题中的 `arg1 = 0x2cb7` 指向 `memcmp` 的延迟绑定槽，`arg2 = 0x22` 指向 `init` 内一段被正常控制流跳过的隐藏代码。于是后续每次调用 `memcmp` 都会进入隐藏比较函数。

隐藏函数仍比较两个 32 位块，但在返回比较结果前还执行：

```c
state ^= *(uint32_t *)arg1;
return *(uint32_t *)arg1 != *(uint32_t *)arg2;
```

这里的 `arg1` 是已经与 PRNG 输出异或后的用户块。若输入正确，该块应等于对应的 `flag_enc` 块，所以每轮比较还会用当前密文块更新下一轮 PRNG 状态。

### 2. 复现状态机并逆变换

`gen_rand()` 把 64 位 `state` 拆成 9、11、13 位三个 LFSR，分别按给定反馈多项式推进，并用组合函数生成 32 位输出。无需从数学上破解它，只需逐句复现程序。

对第 $i$ 块，设进入该轮时的状态为 $S_i$，PRNG 输出为 $R_i$，正确明文为 $P_i$，常量密文为 $C_i$。程序要求：

$$P_i \oplus R_i = C_i$$

因此：

$$P_i = C_i \oplus R_i$$

但下一轮状态不是单纯的 PRNG 内部状态，还要执行隐藏比较函数的更新：

$$S_{i+1}=\operatorname{PRNGState}(S_i)\oplus C_i$$

官方 solver 的核心顺序正是：

```c
uint64_t state = 0xfeedf00ddeadbeef;

for (int i = 0; i < 12; i++) {
    uint32_t r = gen_rand();
    state ^= *(uint32_t *)(enc + i * 4);
    *(uint32_t *)(enc + i * 4) ^= r;
}

printf("%s\n", enc);
```

注意必须先用当前 `flag_enc` 块更新 state，再覆盖该缓冲区为明文；若忽略自定义 `memcmp` 的副作用，从第二块开始所有 PRNG 输出都会错误。

运行复现程序得到：

```text
TSGCTF{h1dd3n_func7i0n_4nd_s31f_g07_0verwr173}
```

## 方法总结

本题用 GOT 自修改隐藏了比较函数的状态副作用。静态看到的“PRNG 异或后 memcmp”并不是完整算法，动态检查调用目标或追踪 GOT 写入才能发现状态还会与每个正确密文块异或。逆向自修改程序时，应把函数的输入、输出和全局副作用都纳入模型；即便不完全理解三组 LFSR 的密码学性质，也可以精确复现状态转移并逐块逆变换。
