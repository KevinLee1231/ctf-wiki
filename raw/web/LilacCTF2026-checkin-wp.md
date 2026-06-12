# CheckIn

## 题目简述

题目是 Python eval jail。白盒源码中可以看到程序创建 `LockedList([False])` 作为 `status`，对用户输入做 IDNA 归一化和正则过滤，然后直接 `eval(user_input)`。如果执行后 `status[0]` 为真且对象 id 未变，就读取 `/flag`：

```python
status = LockedList([False])
status_id = id(status)
user_input = argv[1].encode("idna").decode("ascii").rstrip("-")
...
eval(user_input)

if status[0] and id(status) == status_id:
    print(open("/flag").read().strip())
```

过滤禁止数字、大写字母、下划线、空白、赋值、方括号、引号以及大量关键词，但没有禁止小写字母、点号、括号、`~` 等符号。

## 解题过程

### 白盒解法

核心目标是修改 `status[0]`。`LockedList` 禁止 `__setitem__`，但没有禁止 `pop()` 和 `append()`。由于 `False` 在 Python 中等价于 `0`，`~False == -1`，而非零整数在布尔上下文中为真。因此可以先弹出 `False`，再 append 一个真值：

```python
vars().get(min(dir())).append(~vars().get(min(dir())).pop())
```

这个 payload 的作用是：

1. `vars()` 取得当前局部变量字典。
2. `min(dir())` 取到局部作用域中排序最前的变量名，官方环境中对应 `status`。
3. `pop()` 移除原来的 `False`。
4. `~False` 得到 `-1`。
5. `append(-1)` 使 `status[0]` 在判断中为真。

payload 长度正好满足 60 字节限制。

### 黑盒推导

如果没有 `backup.zip`，也可以从回显推导：

1. 输入 `()` 可看到 `status is still [False]`，说明目标是改变列表内容。
2. Unicode 圈字母经过 IDNA 会还原为 ASCII，可以绕过“禁止数字和大写字母”的表层限制。
3. 扫描可用符号和内置函数，得到 `vars`、`dir`、`min`、`append`、`pop`、`~` 等可用组件。
4. 用 `vars().get(min(dir()))` 定位 `status` 对象，再用 `pop/append` 原地修改。

官方还提到可以用 Unicode 兼容字符缩短 payload，例如 `㏌` 经过 IDNA/NFKC 相关处理可表现为 `in`，但本题不需要这条路径。

## 方法总结

- 核心技巧：不做赋值、不用下标，改用列表原地方法 `pop/append` 修改受保护对象。
- 识别信号：eval jail 中保留 `vars()`、`dir()`、点号和括号，且目标对象是可变容器。
- 复用要点：过滤关键词时要考虑编码归一化；禁止 `__setitem__` 不代表容器不可变，所有 mutating method 都需要纳入攻击面。
