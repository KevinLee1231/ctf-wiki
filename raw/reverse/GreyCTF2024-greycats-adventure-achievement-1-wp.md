# GreyCTF2024 Greycat's Adventure - Achievement 1 WP

## 题目简述

题目给出一款 Unity IL2CPP 游戏，要求把最高分精确改为 `1337420`。正常游玩并不是重点，真正需要识别的是 Unity 构建产物中仍然保留了大量字符串元数据，可以先从元数据直接定位 flag，再用内存扫描验证高分变量确实可被修改。

## 解题过程

游戏数据目录中最值得先检查的是 IL2CPP 元数据文件：

```text
GreyCat's Adventure_Data/il2cpp_data/Metadata/global-metadata.dat
```

直接在整个游戏目录中递归搜索 flag 前缀：

```bash
grep -aR -n "grey{" "GreyCat's Adventure_Data"
```

命中 `global-metadata.dat` 后，再提取其中的可打印字符串：

```bash
strings "GreyCat's Adventure_Data/il2cpp_data/Metadata/global-metadata.dat" \
  | grep -F "grey{"
```

输出中有四个明文 flag。Achievement 1 的目标数值是 `1337420`，因此与 `1337_g4m3r` 语义对应的字符串就是本题答案：

```text
grey{1337_g4m3r_un17y_h4ck3r_12jjsd3}
```

若按题目预期动态验证，可在游戏运行时用内存扫描器搜索当前分数，得分变化后再次筛选变化值，锁定整型变量并写成 `1337420`。这条动态路径解释了成就的触发方式，但静态元数据已经足以恢复 flag。

## 方法总结

Unity IL2CPP 并不等于所有符号和文本都会消失。拿到完整游戏目录时，应先检查 `global-metadata.dat`、资源文件及可打印字符串，再决定是否需要进一步反编译或动态改内存。本题的最短路径是用题目目标数值给四个明文 flag 做语义匹配。
