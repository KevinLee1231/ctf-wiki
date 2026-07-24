# Death Gripes

## 题目简述

附件是一个 Ruby 脚本。模块 `LMAO` 为常量 `A` 到 `Z` 动态建立 lambda；数据区则反复写出 `const_get(...).call.call...`。代码没有传统字符串常量，而是用调用次数编码字符。

## 解题过程

每个 lambda 的核心逻辑是：

```ruby
$A ||= 0
$A = $A.succ
const_get(:A)
```

也就是说，每调用一次就递增该字母对应的全局计数器，再返回同一个 lambda，使 `.call` 可以无限串联。因此无需执行混淆代码，只要按块统计 `.call` 的数量，并把数量解释成 ASCII。

对数据区的 14 个 `const_get` 块计数，得到：

```text
109 99 95 114 105 100 101 95 105 115 95 98 97 101
```

转换为字符：

```python
values = [109, 99, 95, 114, 105, 100, 101, 95, 105, 115, 95, 98, 97, 101]
print(bytes(values).decode())
```

输出：

```text
mc_ride_is_bae
```

最终 flag：

```text
UMDCTF-{mc_ride_is_bae}
```

其 SHA-256 与 README 中的 `e261d9c5c4d5d1afb835e576e73935b5d6ad2f1bcec101e5080be9538c743406` 一致。

## 方法总结

动态语言混淆经常把简单常量藏进语法噪声。先识别 `.call` 链的稳定状态转移，就能把整个程序简化成一元计数编码。静态计数比直接 `eval` 未知数据区更安全，也更容易说明结果为何成立。
