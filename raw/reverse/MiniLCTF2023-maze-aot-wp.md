# MiniLCTF2023 - maze_aot

## 题目简述

题目把一个有向二叉迷宫提前编译进 `maze_walk` 的控制流。每个节点先调用 `maze_step` 取得当前输入位，再通过 `test` 和 `je/jne` 选择两个后继；到达 `maze_final` 后，64 位路径值作为 RC4 密钥解密全局 `maze_flag`。

核心不是在界面中试路，而是从机器码恢复控制流图并求起点到终点的最短路径。

## 解题过程

二进制保留了 `.symtab`，可以直接定位 `maze_walk`、`maze_step`、`maze_final` 和 `maze_flag`。`maze_walk` 中每个基本块只有四种模板：

```text
call test je
call test jne
call test je  jmp
call test jne jmp
```

其中条件跳转和顺序落下/无条件跳转分别对应输入位 0、1 的后继。把块首地址映射为节点编号后做 BFS，路径中每一步选择的边号就是对应 bit；程序按低位在前消费 64 位，所以最终用 little-endian 8 字节作为 RC4 key。

```python
from capstone import Cs, CS_ARCH_X86, CS_MODE_64
from Crypto.Cipher import ARC4
from elftools.elf.elffile import ELFFile
from networkx import DiGraph, shortest_path


def symbol_bytes(elf, sym):
    address, size = sym["st_value"], sym["st_size"]
    for section in elf.iter_sections():
        start = section["sh_addr"]
        if start <= address and address + size <= start + section["sh_size"]:
            off = address - start
            return section.data()[off:off + size]
    raise ValueError("symbol is outside file-backed sections")


with open("maze", "rb") as fp:
    elf = ELFFile(fp)
    symtab = elf.get_section_by_name(".symtab")
    sym = lambda name: symtab.get_symbol_by_name(name)[0]

    walk = sym("maze_walk")
    code = symbol_bytes(elf, walk)
    step_addr = sym("maze_step")["st_value"]
    final_addr = sym("maze_final")["st_value"]
    encrypted = symbol_bytes(elf, sym("maze_flag"))[:-1]

cs = Cs(CS_ARCH_X86, CS_MODE_64)
blocks, current = [], None
final_block = None
for address, _, mnemonic, operand in cs.disasm_lite(code, walk["st_value"]):
    if mnemonic == "nop":
        continue
    if mnemonic == "call" and int(operand, 0) == step_addr:
        current = []
        blocks.append(current)
    elif mnemonic == "call" and int(operand, 0) == final_addr:
        final_block = address
    elif current is not None:
        current.append((address, mnemonic, operand))

starts = [block[0][0] - 5 for block in blocks]  # call 指令为每块开头
starts.append(final_block)
edges = []
for i, block in enumerate(blocks):
    fallthrough = starts[i + 1]
    jumps = [(op, int(arg, 0)) for _, op, arg in block if op in ("je", "jne", "jmp")]
    cond = next((item for item in jumps if item[0] in ("je", "jne")), None)
    direct = next((item for item in jumps if item[0] == "jmp"), None)
    other = direct[1] if direct else fallthrough
    if cond[0] == "je":
        edges.append((cond[1], other))
    else:
        edges.append((other, cond[1]))

index = {address: i for i, address in enumerate(starts)}
graph = [(index[a], index[b]) for a, b in edges]
G = DiGraph()
for node, (zero, one) in enumerate(graph):
    G.add_edge(node, zero)
    G.add_edge(node, one)

path = shortest_path(G, 0, len(starts) - 1)
key, node = 0, 0
for bit_index, nxt in enumerate(path[1:]):
    key |= graph[node].index(nxt) << bit_index
    node = nxt

plain = ARC4.new(key.to_bytes(8, "little")).decrypt(encrypted)
print(f"miniLctf{{{plain.decode()}}}")
```

输出为：

```text
miniLctf{YOU_AR3_$0_GOOD_4T_SOLV1NG_MAZE}
```

也可以用 angr 符号执行，但必须按节点地址和深度剪枝，否则大量回路会造成路径爆炸。

## 方法总结

AOT 迷宫本质是控制流图问题：先识别重复基本块模板，再抽象有向边，最后使用标准最短路。输入位序要从 `maze_step` 的消费方式确认，不能默认按字符串高位在前。符号执行可作为替代，但在结构已高度规则化时，静态抽图更快，也更容易验证每条边的含义。
