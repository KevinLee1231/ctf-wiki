# Gondola

## 题目简述

发布文件是体积很大的 `chal.lua`。仓库构建脚本显示，它并非手写 Lua，而是先把 C++ 编译为 WASI WebAssembly，再用 `wasm2luajit` 转成 LuaJIT：

```sh
clang++-20 --target=wasm32-wasi -O2 -o luawasm chal.cpp
wasm2luajit luawasm > chal.lua
```

底层校验把 32 字节输入分为 8 个 4 字节块。每一块作为 32 位 `FlagKey` 参与大量 64 位加减、异或、移位、循环移位和字节序变换，结果与一个 64 位常量比较。

## 解题过程

### 1. 从生成 Lua 中恢复有效校验

`chal.lua` 前半包含完整 WebAssembly 运行时、内存模型和标准库转换，绝大部分与 flag 无关。应沿最终输入和比较点反向切片，恢复出：

```cpp
for (int i = 0; i < 8; i++) {
    uint32_t key = load_le32(flag + 4 * i);
    uint64_t value = flag_decode_i(flag_encoded[i], key);
    if (value != flag_cmp[i]) fail();
}
```

每个 `flag_decode_i` 互相独立，因此不必让 Z3 同时求解 256 位输入。

### 2. 用 64 位位向量精确翻译

所有计算都有无符号 64 位溢出，普通 Python 整数会得到错误语义。官方 `solve.py` 把每个运算映射到 Z3 BitVec：

```python
rotl_i64 = RotateLeft
rotr_i64 = RotateRight
shr_u64 = lambda a, b: LShR(a, b)

key = BitVec("key", 64)
solver.add(key & 0xffffffff == key)
```

然后逐句翻译恢复出的 Lua/C++ 表达式，并加入：

```python
solver.add(decoded == expected)
```

每个方程只含一个 32 位未知块，求解速度和可诊断性都比整题符号执行更好。

### 3. 处理特殊第四块

官方脚本对第四块直接使用已知前缀结构：

```python
return "sm_1"
```

其余七块由 Z3 模型求出，再按小端序转回 4 字节：

```python
chunk = model.eval(key).as_long().to_bytes(4, "little")
```

拼接 8 块得到完整 32 字节输入。

### 4. 本地验证

使用仓库官方脚本和本地虚拟环境运行，实际输出为：

```text
SEKAI{lua_wasm_1s_very_fun_3hee}
```

该结果来自官方方程求解的本地执行验证，不是根据题名猜测。

## 方法总结

面对“某语言生成的另一种语言”，首先应剥离运行时模板，只保留输入到比较点的数据流。这里 Lua 只是 WebAssembly 语义的载体，真正约束仍是 8 个互相独立的 32 位位向量方程。

翻译时最重要的是位宽和移位语义：逻辑右移必须用 `LShR`，加减乘要保持 64 位溢出，输入块按小端序解释。任何一处改成 Python 无限精度或算术右移，都会让方程看似合理却无解。
