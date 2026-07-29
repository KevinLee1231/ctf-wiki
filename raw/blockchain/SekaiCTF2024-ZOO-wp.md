# ZOO

## 题目简述

`ZOO` 的 fallback 用手写 EVM assembly 解析一串 `ADD`、`EDIT`、`DEL` 指令，在内存中维护最多八个本地动物指针，最后通过内存中的函数指针调用 `commit(local_animals)`。合约构造时调用 `_pause()`，正常路径无法进入带有 `whenNotPaused` 的 `commit`。

解析器的关键问题是 `DEL`：

```solidity
let copy_size := sub(0x100, mul(0x20, idx))
mcopy(offset, add(offset, 0x20), copy_size)

let animals_count := mload(local_animals)
animals_count := sub(animals_count, 1)
mstore(local_animals, animals_count)
```

它按固定大小向前搬移指针，越过本地数组的实际边界；重复删除同一元素还会让计数下溢。`EDIT_TYPE` 随后对取出的指针执行 `mstore(add(temp, 0x20), new_type)`，形成对周边内存的定向改写。

## 解题过程

### 建立内存布局

官方求解器根据本次 fallback 的稳定布局使用以下偏移：

| 位置 | 偏移 |
|---|---:|
| 函数指针表 | `0x80` |
| `local_animals` | `0x300` |
| 第一个本地动物指针 | `0x320` |
| 第一次动态分配 | `0x400` |
| 伪造动物结构 | `0x460` |
| 跳过暂停检查后的目标位置 | `0x323` |

先添加一个伪造动物。其名字区域保存后续 `commit` 要解析的结构，其中索引通过动态数组 storage 基址反推，使最终存储访问落到目标槽位：

```solidity
uint256 target_idx =
    uint256(int256(-1)) - uint256(keccak256(abi.encode(2)));
target_idx = target_idx / 2 + 1;

bytes memory fake_animal = abi.encodePacked(
    uint256(target_idx),
    uint256(0x20),
    hex"41414141"
);
```

### 用删除和编辑改写函数指针

利用链按如下顺序移动和改写内存指针：

1. `ADD` 在索引 `0` 放入伪造动物，并令其类型字段指向第一次分配位置 `0x400`。
2. 删除索引 `0`，使第七个槽位被越界搬移为 `0x400`。
3. 编辑索引 `7` 的类型，把该指针改为函数表偏移 `0x80`。
4. 再删除索引 `0`，令第七个槽位指向函数表。
5. 编辑索引 `7`，把函数入口由正常的 `commit` 改为 `0x323`，从暂停检查之后继续执行。
6. 把索引 `6` 的类型改为 `0x300`，再删除一次，使索引 `7` 指向 `local_animals`。
7. 把该位置改为伪造结构偏移 `0x460`。
8. 三次向索引 `1` 添加动物，抵消此前三次删除造成的计数下溢，并使 `commit` 只处理预期结构。
9. 将总 calldata 填充到 `0x200` 字节，使上述相对偏移保持稳定。

最终调用由拼接后的二进制指令流一次完成：

```solidity
(bool success, bytes memory out) = address(zoo).call(call);
require(success);
assert(setup.isSolved());
```

被篡改的函数指针跳过 `whenNotPaused` 检查，`commit` 再使用伪造动物索引完成目标 storage 写入，使 `isSolved()` 返回 `1`。官方解法给出的 flag 为：

```text
SEKAI{super-duper-memory-master-:3}
```

## 方法总结

- 核心技巧：利用 assembly 数组删除的越界 `mcopy` 和计数下溢，把数组元素编辑能力扩大为函数指针改写，再伪造提交结构完成 storage 写入。
- 识别信号：手写解析器同时维护长度、裸指针和固定大小内存搬移；函数指针与用户可控动态数据位于相邻内存区域。
- 复用要点：分析内联 assembly 时应画出逐字内存布局，并对每次 `mcopy` 的源、目标、长度和重叠方向做状态跟踪；高级 Solidity 的边界检查不会保护手写的 EVM 内存管理。
