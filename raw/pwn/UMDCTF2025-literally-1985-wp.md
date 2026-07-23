# UMDCTF 2025 - literally-1985

## 题目简述

本题沿用 `literally-1984` 的 V8 JIT 漏洞：TurboFan 把 `2 + 2` 错误折叠为
`5`，同时边界检查强化被恒假的 C++ 条件关闭。新增补丁把
`Shell::CreateGlobalTemplate` 中大部分 d8 专用全局对象注册代码注释掉，意图限制
调试和辅助接口：

```cpp
Local<ObjectTemplate> global_template = ObjectTemplate::New(isolate);
/*
  global_template->Set(...);
  ...
*/
return global_template;
```

这只是缩减 shell 功能，没有修复编译器对数值范围的错误认识，也没有移除标准
JavaScript 的 TypedArray、ArrayBuffer 和 WebAssembly。

## 解题过程

### 1. 重新触发优化差异

利用仍从下面的表达式开始：

```javascript
let x = 2;
let bad = x + 2;
bad = bad - 4;
```

未优化代码得到 `bad = 0`；TurboFan 优化版本把 `x + 2` 变成 `5`，得到
`bad = 1`。在同一优化函数中布置 `arr2 = [1, 1, 1]` 和
`arr1 = [1.1, 1.2, 1.3]`，再执行：

```javascript
arr1[bad * 2] = 21;
arr2[bad * 4] = 10000;
```

第二次写入在优化版本中越过 `arr2`，把相邻 `arr1` 的长度改成 10000。补丁删除的
d8 全局函数并不参与这一步，因此漏洞原语与前一题相同。

### 2. 恢复地址泄漏与任意读写

扩大后的 `arr1` 可以访问相邻对象。官方脚本从越界槽位泄漏压缩对象指针，并从
另一个槽位恢复 V8 isolate/cage 的高 32 位。把两部分拼接后，就能定位目标对象的
完整地址。

随后创建一个 `ArrayBuffer`，利用越界写修改它的 backing store 指针：

```javascript
let victimb = new ArrayBuffer(0x800);

function point_buffer_at(addr) {
    arr1[11] = addr.i2f();
}
```

指向任意地址后，基于该缓冲区的 `DataView` 和 TypedArray 就构成任意读写。这里
只使用标准语言与 Wasm 功能，不依赖被删除的 `version`、调试辅助或其他 d8 builtin。

### 3. 从 Wasm 转入 JIT shellcode

脚本实例化一个小型 WebAssembly 模块以获得可执行 jump table，同时预热一个返回
多组浮点常量的 `f2()`。这些常量在 JIT 机器码中编码成
`execve("/bin//sh", ...)` shellcode。

由于补丁和对象分配变化，本题中 `f2` 的压缩代码指针偏移从前一题的
`0x3800cc` 变为 `0x1400f4`。利用读取该处的 JIT 代码地址，加上常量机器码的内部
偏移，再把 Wasm jump table 的首项改成 shellcode 地址。调用 Wasm 导出函数后获得
shell，并读取：

```text
UMDCTF{next_time_i_should_probably_disable_the_d8_builtins_lesson_learned}
```

## 方法总结

本题说明“删掉利用时常见的调试接口”与“消除漏洞利用能力”是两回事。只要 JIT
错误仍能产生越界，标准对象布局就足以构造地址泄漏和任意读写；只要运行时仍提供
可执行 JIT/Wasm 代码，攻击者就能寻找新的控制流落点。真正的修复应恢复正确的常量
折叠与边界检查，而不是依赖 d8 外壳功能裁剪。版本变化还会改变对象和 JIT 偏移，
所以应从当前二进制重新测量，而不能机械复用旧常量。
