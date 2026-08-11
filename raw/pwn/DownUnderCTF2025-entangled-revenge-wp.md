# entangled revenge

## 题目简述

这是 `entangled` 的 revenge 版本，仍为修改过的 V8 `d8` worker/isolate 题。题目保留了受限 d8 环境和 `Flag` 对象：其第 0 个 internal field 初始为 `0xdead`，只有改成 `0x1337` 时 `initialize()` 才把 flag 放进第 1 个 field。

与原题不同，部署 Dockerfile 将 flag 放在随机文件名，并通过 `FLAG` 环境变量传给 `FlagInitialize()`；README 同时说明原版部署存在非预期解，且 diff 已修正、d8 已重新构建。因此不能依赖已知 `flag.txt` 文件名或原题工件不一致的偶然行为，必须利用 V8 漏洞读取 `Flag` 内部存放的字符串。

## 解题过程

### 错误的跨 isolate 地址传递

新增 API 的发送端执行：

```cpp
uint32_t addr = static_cast<uint32_t>(
    Cast<i::HeapObject>(*arg)->address());
worker->PostMessageEntangle(addr);
```

接收端却以 `sandbox->base() + address` 直接构造 `HeapObject`。队列中只有截断后的地址，既没有 V8 的正常 structured clone，也没有对接收 isolate 的对象归属检查，更没有把对象纳入正确的 GC 生命周期。于是主 isolate 中已可回收的数组在 worker 中仍可被当作有效对象访问。

官方 solver 的布局策略是：主线程把大浮点数组 `m` 交给 worker 后触发两次 minor GC；再批量分配保存哨兵浮点、`ArrayBuffer`、浮点数组和对象数组的组合。worker 对悬垂的 `stolen` 数组扫描，寻找被新布局覆盖的哨兵槽，解析邻近 metadata 后把候选数组编号与地址信息发回主线程。主线程利用尾部相邻写重新定向该混叠数组；worker 再识别 `ArrayBuffer` 头部并篡改可控字段，使主线程的 DataView 覆盖所需 V8 sandbox heap 区域。

### 从对象地址到 flag 字符串

主线程以对象数组/浮点数组的元素表示差异实现 `getAddressOf`，获得压缩对象地址。随后对 `Flag` 做两次有针对性的 DataView 操作：

```javascript
let f = new Flag();
let addr = getAddressOf(f);
dv.setUint32(addr + 0x10, 0x1337 << 1, true); // field 0 = 0x1337
f.initialize();

let string = dv.getUint32(addr + 0x18, true); // field 1
dv.setUint32(getAddressOf(test) + 0xc, string, true);
print(test.a);
```

第一处写入把 field 0 改为 Smi 形式的允许值，`initialize()` 因而通过检查并根据 `getenv("FLAG")` 打开随机文件名；第二处读取取回 field 1 的字符串 tagged pointer，把它放入普通对象属性后交给 `print`。这条链与随机文件名无关，因而适用于 revenge 部署。

### 验证

官方 solver 的末行 `print(test.a)` 是唯一需要的结果验证：它表明 condition field 已修改、`FlagInitialize` 已读取环境变量给出的文件，且字符串对象已被恢复为可打印属性。本文没有运行题目 d8 或 solver，结论来自 revenge 的 challenge diff、Dockerfile 和随附官方 exploit。

## 方法总结

- 核心技巧：跨 isolate 原始地址传递破坏了 V8 的对象归属与 GC 边界；通过 GC 后的重分配把问题升级为可控 heap 混叠。
- 识别信号：挑战修复了可预测文件路径但保留引擎级读写 primitive 时，应转而攻击数据流终点，例如存储 flag 的 internal field，而不是猜文件名。
- 复用要点：压缩指针环境下先确认地址是 sandbox 偏移还是完整指针；对象头和 internal field 的偏移必须以提供的 d8 构建为准，不能套用其他 V8 版本。
