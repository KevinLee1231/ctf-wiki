# chainflag

## 题目简述

Solidity 的 `private` 只限制其他合约通过自动 getter 访问变量，不会加密链上存储。源码中 `xornum` 位于槽 4，动态数组 `flag` 的长度位于槽 5；数组元素从 `keccak256(uint256(5))` 对应的槽开始连续存放。每个元素的前 6 字节与 `xornum` 的低 48 位异或后就是 flag 片段。

## 解题过程

状态变量按声明顺序布局：`chainaddr` 在槽 0，三个 `uint` 在槽 1 到 3，`xornum` 在槽 4，动态数组头在槽 5。动态数组本体不紧跟槽 5，而从：

$$
base=\operatorname{keccak256}(\operatorname{uint256}(5))
$$

开始。`bytes32[]` 每个元素占一个完整槽，因此第 $i$ 项在 `base+i`。

下面脚本只需替换当前 RPC 和题目实例地址：

```python
from web3 import Web3

w3 = Web3(Web3.HTTPProvider("http://rpc-endpoint"))
address = Web3.to_checksum_address("0xChallengeAddress")

def read_slot(slot: int) -> bytes:
    return bytes(w3.eth.get_storage_at(address, slot))

xornum = int.from_bytes(read_slot(4), "big")
length = int.from_bytes(read_slot(5), "big")
assert length == 5

base = int.from_bytes(Web3.keccak((5).to_bytes(32, "big")), "big")
key = (xornum & ((1 << 48) - 1)).to_bytes(6, "big")

plain = bytearray()
for i in range(length):
    stored = read_slot(base + i)
    encrypted_block = stored[:6]  # bytes6 转 bytes32 后位于高位
    plain.extend(a ^ b for a, b in zip(encrypted_block, key))

print(plain.rstrip(b"\x00").decode())
```

构造函数对五个 `bytes6` 片段都使用相同 `xornum`，所以读取一次槽 4 后可解开全部 30 字节并直接拼接。

## 方法总结

链上状态对任何节点都是公开的，`private` 不是保密机制。读取动态数组时必须根据变量槽号、元素静态大小和 `keccak256` 布局计算真实位置；固定长度 bytes 在更宽 bytes 类型中按左对齐、右侧补零，也要在提取时正确处理。
