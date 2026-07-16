# Berkeley

## 题目简述

题目给出一个 64 位 ELF，使用 libbpf-bootstrap 框架加载 eBPF 程序。表面上用户态 `check_flag()` 会对输入做一次诱饵校验，但真正校验逻辑在内核空间的 uprobe/uretprobe 中完成：程序在 `check_flag` 入口读取用户输入，在返回时继续变换并与 256 字节密文比较。

核心障碍是识别 eBPF 自拦截机制，提取嵌入 BPF 程序中的 `key`、`cipher` 和位掩码数组，再逆向双层置换表。

## 解题过程

### 关键观察

主程序加载 BPF 后把两个程序挂到自身 `check_flag()`：

```text
check_flag(user_input)
  -> uprobe 入口：读取输入，生成 event->output[0..255]
  -> 用户态 check_flag：使用 fake_cipher 做诱饵比较
  -> uretprobe 返回：二次变换 output，并与真正 cipher 比较
```

用户态比较目标 `fake_cipher` 不是正确路径。真正公式可整理为：

```text
cipher[i] = key[ key[ flag[i/8] ^ ~(flag[i/8] + arr[i%8]) ] ^ key[i] ]
```

其中 `arr = [0x80,0x40,0x20,0x10,0x08,0x04,0x02,0x01]`，`key` 是 256 字节置换表，因而存在逆映射。

### 求解步骤

从二进制中提取关键数据：

```text
key    : 256 bytes
cipher : 256 bytes
arr    : 8 个 bit mask
```

先构造 `key_inv`，逐层逆回去：

```python
key_inv = [0] * 256
for i, v in enumerate(key):
    key_inv[v] = i

# 逆 uretprobe 的第二层变换
step1 = [key_inv[cipher[i]] ^ key[i] for i in range(256)]

# 逆 uprobe 的 key 查表
target = [key_inv[x] for x in step1]

arr = [0x80, 0x40, 0x20, 0x10, 0x08, 0x04, 0x02, 0x01]
flag = bytearray()
for j in range(32):
    block = target[j * 8:(j + 1) * 8]
    for ch in range(256):
        if all((ch ^ (0xff ^ ((ch + arr[k]) & 0xff))) == block[k] for k in range(8)):
            flag.append(ch)
            break

print(flag.decode())
```

恢复结果：

```text
71c2ac98ac8d99a2e8a95111449a7393
```

## 方法总结

- 核心技巧：绕过用户态诱饵逻辑，分析 eBPF uprobe/uretprobe 中的真实校验。
- 识别信号：ELF 中出现 libbpf-bootstrap、`attach_uprobe`、BPF map 和用户态校验矛盾时，要检查嵌入 BPF 程序。
- 复用要点：置换表必须先验证是否为双射；有逆表后逐层反解，比尝试模拟用户态 fake check 更可靠。
