# entangled

## 题目简述

题目提供经过修改的 V8 `d8`，服务仅接收一段 Base64 编码 JavaScript 并执行。常见的 `read`、`write`、`load`、`os` 与 `d8` 全局接口被删除；可见的目标对象是 `Flag`，它有两个 internal fields：构造函数把 field 0 设为 `0xdead`，只有 field 0 被改成 `0x1337` 时，`initialize()` 才会读取 `flag.txt` 并把 flag 字符串放入 field 1。

漏洞位于新增的 `Worker.postMessageEntangle()` / `getMessageEntangle()`：发送端把 `HeapObject::address()` 强制截断为 `uint32_t` 放入队列，接收端仅以自己的 sandbox base 加回该值就构造 `HeapObject` handle，没有序列化、对象所属 isolate 验证或 GC 保活。这把一个 isolate 的对象原始引用错误地暴露给另一个 isolate，形成跨 isolate 的悬垂/混叠 heap 引用。

## 解题过程

### 用 worker 制造可控 heap 混叠

官方 solver 在主 isolate 创建一个填满浮点数的大数组 `m`，通过 `postMessageEntangle(m)` 把其截断地址交给 worker，再让 worker 调用 `getMessageEntangle()` 取得 `stolen`。主线程连续触发 minor GC，并布置一组保活对象：带哨兵值的浮点数组、不同大小的 `ArrayBuffer`、浮点数组和对象数组。GC 后，worker 仍持有未被正常追踪的跨 isolate 引用；其所见布局会与主线程新分配对象交叠。

worker 扫描 `stolen` 的槽位，寻找不再等于哨兵 `0x53d0000053d` 的位置，从相邻槽恢复主线程候选数组的索引、map 和 backing-store 元数据。它把这些信息通过普通 `postMessage` 传回主线程；主线程据此在相邻的保活数组尾部写入伪造元数据，重新定向 worker 的混叠数组。

### 升级为 DataView 读写并解锁 Flag

重定向后，worker 继续扫描并识别 `ArrayBuffer` 的特征字。官方脚本修改该对象的长度/相关头部槽，并复制已知 backing-store 指针，随后通知主线程。主线程的 `DataView` 因此可按 sandbox 内偏移读写 V8 heap：对象数组加浮点数组组合提供 `getAddressOf`，返回目标对象的压缩地址。

利用最后不需要代码执行。创建 `Flag` 后，用 DataView 将其 field 0 改为已标记的 Smi `0x1337 << 1`，调用 `initialize()`，再读取 field 1 中字符串的压缩地址，并把它写入普通对象属性槽：

```javascript
let f = new Flag();
let flag_addr = getAddressOf(f);
dv.setUint32(flag_addr + 0x10, 0x1337 << 1, true);
f.initialize();

let string = dv.getUint32(flag_addr + 0x18, true);
dv.setUint32(getAddressOf({a: {}}) + 0xc, string, true);
```

将该字符串 tagged pointer 放入对象属性后，`print(test.a)` 即由 V8 正常打印 flag。扫描使用的具体槽位会随 heap 布局变化，官方 solver 用哨兵和候选范围而非固定绝对地址提高成功率。

### 验证

官方 `solve.js` 的成功终点是 `Flag.initialize()` 填充 field 1，随后 `print(test.a)` 打印该字符串。服务侧只对 Base64 长度设上限，运行脚本会把解码的 JavaScript 写入临时文件再交给 d8。本文依据 challenge diff、部署脚本与官方 solver 整理，未执行 d8 或 exploit。

## 方法总结

- 核心技巧：跨 isolate 传递未经序列化且未被 GC 保活的原始 heap 引用，将隔离边界降级为可控的 heap 混叠。
- 识别信号：V8/JS 引擎题若新增 worker 消息 API，却直接传递压缩地址或通过 `FromAddress` 重建对象，应检查 isolate 归属、sandbox base、生命周期和 GC root。
- 复用要点：稳定利用通常先以哨兵定位重用布局，再做有限的 sandbox heap 读写；很多 Flag API 的条件字段可被直接篡改，无需进一步逃逸到原生 shell。
