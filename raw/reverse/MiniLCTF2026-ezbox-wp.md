# ezbox

## 题目简述

附件是一个 PyInstaller 打包的 Linux ELF，内部是递归推箱游戏。主关为 `h10`，数字方块 `k` 在未解锁时会进入子关 `hk`，子关完成后才变成可推动方块。递归树共有 $2^{10}=1024$ 个上下文，程序要求完成所有上下文后才尝试解密 flag。

决定性缺陷是：完成哈希不包含方块的实际移动过程，只依赖上下文路径和关卡中固定的目标格坐标。因此无需真正完成 1024 个推箱实例，直接重建哈希集合就能派生 XXTEA 密钥。

## 解题过程

### 解包与定位校验链

用 `pyinstxtractor` 解包后可得到 `main.pyc`、`game.pyc`、`levels.pyc`、`flag.pyc` 以及原生扩展 `_core.so`。打包环境是 Python 3.12；阅读字节码时需使用匹配版本，但解包出的 `.pyc` 也可由同版本 Python 直接导入。

`Game.complete_level()` 的核心逻辑可等价写为：

```python
goals = sorted(
    f"{x},{y}" for (x, y), cell in terrain.items() if cell in "=_"
)
goals_str = ";".join(goals)
completed_hashes[context_path] = hashlib.sha256(
    f"{context_path}|{goals_str}".encode()
).hexdigest()
```

密钥派生则是将所有上下文按路径字典序排列，拼接 64 字符的 SHA-256 十六进制摘要，再做一次 SHA-256：

```python
combined = "".join(completed_hashes[p] for p in sorted(completed_hashes))
key = hashlib.sha256(combined.encode()).digest()[:16]
```

`try_get_flag()` 只检查 `completed_hashes` 是否达到 1024 项，并没有使用声明的 `EXPECTED_TOTAL_STEPS`。`_core.so` 内置 32 字节密文，把上述 16 字节密钥按小端序解释为 4 个 `uint32_t`，然后用常量 `0x9E3779B9` 执行 XXTEA 解密。

### 直接重建 1024 个上下文

上下文树的递归定义是：当前层级为 $n$ 时，依次递归进入 $0,1,\ldots,n-1$ 层。`h10/1/0` 这类上下文使用最后一段选择关卡文件，即 `h0.txt`；根节点 `h10` 使用 `h10.txt`。官方直接解法的关键部分如下：

```python
from levels import collect_all_context_paths, load_level_file
from flag import derive_key, hash_level_state
from _core import decrypt


def file_for_context(context: str) -> str:
    last = context.rsplit("/", 1)[-1]
    return last if last.startswith("h") else f"h{last}"


completed = {}
for context in sorted(collect_all_context_paths()):
    terrain, _, _ = load_level_file(file_for_context(context))
    goals = sorted(
        f"{x},{y}" for (x, y), cell in terrain.items() if cell in "=_"
    )
    completed[context] = hash_level_state(context, ";".join(goals))

key = derive_key(completed)
print(decrypt(key))
```

运行后得到：

```text
miniL{EZ_Hano1_ez_s1gn1n_r1ght?}
```

另一条可行路线是直接导入 `Game`，按“进入未完成数字块 → 递归解子关 → 把已解锁方块推到 `_` 目标 → 玩家走到 `=`”的顺序自动游玩。这能验证游戏状态机，但不如直接重建哈希简洁。

## 方法总结

- 核心技巧：解包 PyInstaller，追踪 Python 与原生扩展之间的密钥传递，再利用“完成哈希只依赖固定关卡数据”的设计缺陷绕过整个游戏。
- 识别信号：题目要求大量递归关卡，但校验值却只取路径和静态地形，说明可以直接构造“已完成”状态。
- 复用要点：不要只逆主程序或 `.pyc`；密文和解密实现可能被拆到 `.so` 扩展中。同时要检查声明的限制是否真的进入最终校验；本题的步数常量就是无效条件。
