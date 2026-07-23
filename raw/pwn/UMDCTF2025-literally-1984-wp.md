# UMDCTF 2025 - literally-1984

## 题目简述

题目提供了 V8 提交 `c963fb98a204005df30553bec7bbbe1997e0ab5f`、构建参数、
补丁和定制 `d8`。补丁故意让 TurboFan 把常量表达式 `2 + 2` 优化成 `5`：

```cpp
if (m.left().Is(2) && m.right().Is(2)) {
    return ReplaceInt32(5);
}
```

它还把边界检查强化条件改成 C++ 编译期恒假的 `2 + 2 == 5`，使
`kAbortOnOutOfBounds` 不再启用。由解释执行与优化后代码之间的数值差异，可以制造
数组越界，最终逃逸 JavaScript 沙箱并执行本机 shellcode。

## 解题过程

### 1. 制造只在优化后出现的索引

官方利用把常量 `2` 先放进变量，避免前端过早把整个表达式折叠掉：

```javascript
function foo() {
    let x = 2;
    let bad = x + 2;
    bad = bad - 4;

    let arr2 = [1, 1, 1];
    let arr1 = [1.1, 1.2, 1.3];

    arr1[bad * 2] = 21;
    arr2[bad * 4] = 10000;
}
```

解释执行时 `bad` 为 `0`，所有访问都在正常位置；TurboFan 优化后却把
`x + 2` 当成 `5`，所以 `bad` 变为 `1`。`arr2[4] = 10000` 越界写到相邻
`arr1` 的长度字段，使一个只有 3 个元素的数组获得约 10000 的伪造长度。
大量预热调用让该函数稳定进入优化版本。

### 2. 从越界数组建立任意读写

扩大后的 `arr1` 能读取相邻对象布局。利用先泄漏一个对象中的压缩指针，再从
越界位置取得 V8 isolate 指针的高 32 位，拼出指针压缩 cage 内的完整地址。
之后修改相邻 `ArrayBuffer` 的 backing store：

```javascript
let victimb = new ArrayBuffer(0x800);
let view = new DataView(victimb);

// arr1[11] 与 victimb 的 backing store 重叠
arr1[11] = target_address.i2f();
```

每次把 `arr1[11]` 改成目标地址后，基于 `victimb` 创建的 `Int32Array`、
`Float64Array` 或 `DataView` 就会在该地址读写，从而得到进程级任意读写。

### 3. 劫持 WebAssembly 跳转表

利用创建一个极小的 WebAssembly 实例，使 V8 分配可执行代码和跳转表；同时反复
调用 `f2()`，让包含若干精心选择的 `Float64` 常量的函数被 JIT 编译。这些常量在
机器码中排列为 `/bin//sh` 的 `execve` shellcode。

官方脚本使用该构建下的压缩指针偏移 `0x3800cc` 找到 `f2` 的 JIT 代码指针，
再加 `0x24d` 定位到常量对应的 shellcode 起点。最后把 WebAssembly jump table
首项改为该地址：

```javascript
arr1[11] = jump_table_addr.i2f();
new DataView(victimb).setFloat64(0, jit_addr.i2f(), true);
f();
```

调用导出的 Wasm 函数后，控制流进入 shellcode，读取 flag：

```text
UMDCTF{W4r_1s_p3ace_Fre3dom_1s_slavery_Br4inrot_is_5trength}
```

## 方法总结

本题的完整链条是“错误常量折叠 → 优化版本专属 OOB → 扩大数组长度 →
伪造 ArrayBuffer backing store → 任意读写 → 劫持 Wasm 跳转表”。分析 JIT 题时，
需要分别验证解释器、基线编译器和优化编译器对同一值的认识；一旦类型或范围假设
不一致，原本被消除的边界检查就会变成安全漏洞。利用中的对象偏移和 JIT 代码偏移
依赖指定 V8 提交及构建参数，不能直接照搬到其他版本。
