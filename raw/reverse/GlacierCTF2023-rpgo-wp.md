# GlacierCTF2023 - rpgo

## 题目简述

附件是一个 Go 编写的文字 RPG，但地图输出被注释。玩家要穿过泥地、击败怪物、取得镐并凿开墙，到达终点 `F`。程序把所有命令不带空格地拼接为 `done_moves`，取其 MD5 十六进制字符串作为 AES-CBC 密钥，解密内置密文。

## 解题过程

源码中的 $10\times10$ 地图可直接恢复：`S` 是起点，`X` 是泥地，`M` 是怪物，`P` 是镐，`W` 是可挖墙。进入泥地会令下一条移动命令只消耗回合而不移动；怪物有 3 点生命，未先 `brace` 会被向左击退；只有站到镐附近再 `pickup` 才能执行 `mine`。

满足状态约束并到达终点的命令序列为：

```text
go down
go down
go down
go right
go down
go down
go brace
go fight
go fight
go fight
go right
go right
go right
go up
go up
go left
go pickup
go right
go down
go down
go right
go down
go right
go mine
go right
go right
```

程序实际哈希的字符串是：

```text
downdowndownrightdowndownbracefightfightfightrightrightrightupupleftpickuprightdowndownrightdownrightminerightright
```

其 MD5 十六进制结果恰为 32 字节 ASCII，可直接作为 AES-256 密钥。程序从内置密文前 16 字节取 IV，CBC 解密后输出：

```text
gctf{Pl34se_d0nt_buY_Our_g4m3s!}
```

## 方法总结

此题的核心不是盲玩，而是恢复隐藏状态机：地图符号、延迟回合、战斗击退、拾取与挖掘都影响最终命令串。由于命令串同时充当解密密钥，路径“能走通”还不够，必须与程序期望的精确动作序列一致。
