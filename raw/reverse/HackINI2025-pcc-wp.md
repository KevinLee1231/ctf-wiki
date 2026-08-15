# pcc

## 题目简述

这是一个带 `ptrace` 反调试和多段无效函数的 C++ 加密程序。它从看似复杂的字符序列生成 16 字节 AES-128-ECB 密钥，并把输入零填充后写入 `output.enc`。关键弱点是 `std::set` 会对字符去重并排序，密钥中只有最前面的两个随机字节未知，因此总搜索空间只有 $2^{16}=65536$。

## 解题过程

### 去掉 C++ 容器造成的视觉干扰

`key_gen` 先把大量字符压入 `vector<uint8_t>`，其中包含重复的 `this_is_a_set_d3dup`、`flag`、`password` 等片段。随后执行：

```cpp
set<uint8_t> s;
for (uint8_t byte : map) {
    s.insert(byte);
}

deque<uint8_t> key_deque(s.begin(), s.end());
```

`set` 的结果与原字符顺序和重复次数都无关：它只留下不同字节，并按数值升序排列。确定性部分为：

```text
13_adfghloprstuw{}
```

程序又生成两个随机字节，并逐个 `push_front`：

```cpp
vector<uint8_t> random_prefix = generate_random_bytes(2);
for (uint8_t byte : random_prefix) {
    key_deque.push_front(byte);
}
```

第二个随机字节最终排在第一个随机字节之前，但求解时枚举全部两字节组合即可，无须恢复其生成顺序。AES 只取队列前 16 字节，所以密钥结构为：

```text
[未知字节 0][未知字节 1]13_adfghloprst
```

### 穷举 16 位随机前缀

加密采用 AES-128-ECB、关闭 OpenSSL 自动填充，并把明文用 `\x00` 补到 16 字节倍数。对 `output.enc` 枚举两字节前缀，以已知 flag 格式校验结果：

```python
from itertools import product
from Crypto.Cipher import AES

with open("output.enc", "rb") as f:
    ciphertext = f.read()

key_tail = b"13_adfghloprst"

for prefix in product(range(256), repeat=2):
    key = bytes(prefix) + key_tail
    plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
    if plaintext.startswith(b"shellmates{"):
        print(plaintext.rstrip(b"\x00").decode())
        break
```

得到：

```text
shellmates{4re_yOu_th4t_g0Od_iN_c++}
```

仓库内的 `flag.txt` 与该明文一致。

## 方法总结

反调试代码和冗长的字符构造并未增加密码强度。逆向 C++ 程序时，必须按容器语义化简数据流：`set` 意味着去重加排序，`deque::push_front` 会反转逐次插入的前缀顺序。最终真正未知的只有两个字节，可用已知明文前缀快速验证。不要把源码长度或控制流噪声误当成密钥熵。
