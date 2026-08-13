# GreyCTF2022 - Angry Robot

## 题目简述

服务连续给出许多由模板随机生成的小型校验程序，逐个手工逆向来不及。每个程序都读取一段输入，对每个字节调用同一个 `scramble` 函数两次并与两组期望数组比较，适合用 angr 自动提取结构并逐字节求解。

## 解题过程

官方 `solve.py` 先用 `CFGFast` 找到 `main`、检查函数和 `scramble` 调用点。它 hook `fgets` 取得输入长度，再在循环前 hook 一次，从栈帧中读出 `expect1`、`expect2` 两个数组，避免让符号执行在循环中爆炸。

```python
project.hook_symbol('fgets', FgetsSim())
project.hook(loop_addr - 2, LoopSim())
project.factory.simgr().explore(find=loop_addr)
```

随后每个字符独立建立 8 位符号变量 $u_i$，约束为

$$scramble(u_i,i)=expect1_i,$$

$$scramble(u_i,expect1_i)=expect2_i,$$

并限制到可打印 ASCII。这样每次只解一个很小的约束系统。把每个二进制的答案按服务顺序提交，最终 flag 文件为：

```text
grey{A11_H4il_SkyN3t}
```

仓库 README 写成了 `flag{...}`，但实际服务 `flag.txt` 使用 `grey{...}`，此处以服务文件为准。

## 方法总结

批量逆向的重点是找稳定结构并把昂贵部分 hook 掉。不要让 angr 从入口盲跑所有循环；先用 CFG 确认调用关系，再提取常量、按彼此独立的字节建立约束，速度和稳定性都会显著提升。
