# DownUnderCTF 2022 optimal vee ate Writeup

## 题目简述

题目基于 V8 commit `1e98dd917ec4e8234174eea326fe8cf8fa283d2f`，并给出一份修改 `Array.prototype.concat` 快速路径的 Torque patch。服务用 d8 执行一行 JavaScript，启动参数包含：

```sh
--no-wasm-write-protect-code-memory
```

patch 在右侧数组长度为 1 时调用 `AppendToFastJSArray`。当唯一元素是 `null` 或 `undefined`，函数没有扩容或复制 backing store，只把左数组的 `length` 增加 1，而且返回原数组本身。

## 解题过程

### 制造越界数组

创建只有一个 double 元素的数组，反复拼接 `[undefined]`：

```javascript
var oob_arr = [1.1];
for (let i = 0; i < 999; ++i) {
  oob_arr.concat([undefined]);
}
console.assert(oob_arr.length === 1000);
```

逻辑长度变成 1000，但 elements backing store 仍只容纳原来的一个元素，于是后续索引可以读写相邻 V8 堆对象。提前顺序分配 `oob_arr`、double 数组和对象数组，可在固定的 OOB 索引读到各自 map；官方布局中 double array map 位于索引 5，对象数组 map 位于索引 14。

### 从 map 置换得到 `addrof` 与 `fakeobj`

V8 开启指针压缩，对象引用位于 64 位槽的低 32 位。把对象数组的 map 临时改成 double array map，读取元素即可得到对象压缩地址，形成 `addrof`；反向把 double array 的 map 改成 object array map，则可把数值中的压缩指针解释为对象，形成 `fakeobj`。

```javascript
function addrof(obj) {
  obj_arr[0] = obj;
  // 通过 oob_arr[14] 暂时把 obj_arr.map 换为 f64_map
  let ptr = float_to_uint(obj_arr[0]) & 0xffffffffn;
  // 恢复 obj_map
  return ptr - 1n;
}
```

再在普通 double 数组内容中伪造一个 JSArray header，控制其 elements 指针和 length，就能把 `fake[0]` 变成任意 64 位读写。关键是保留 map、properties 等合法字段，只改 elements 指向 `target - 8 + 1`，其中 `+1` 是 V8 heap object tag。

### 改写 WASM RWX 页

创建一个最小 WebAssembly 实例。官方构建下，实例对象偏移 `0x68` 保存其可执行页地址：

```javascript
var rwx_page = arb_read(addrof(wasm_instance) + 0x68n);
```

随后创建 `ArrayBuffer`，用任意写把其 backing store（该构建偏移 `0x1c`）改到 RWX 页，再通过 `DataView.setUint8` 写入执行 `/bin/sh` 的 x86-64 shellcode。调用 WASM 导出函数时，控制流进入已覆盖的代码页。

服务端 `worker.js` 只执行一次 `readline()`，所以最终 exploit 必须压成单行。进入 shell 后读取随机前缀的 flag 文件：

```text
DUCTF{1_gu3S5_0pt1MiZ4ti0n_1s_h4Rd3R_tH4n_1_Th0uGhT}
```

## 方法总结

漏洞根因是只修改 JSArray 的逻辑长度，却没有同步 elements 容量。V8 利用链依次为 OOB → map 泄漏与置换 → `addrof/fakeobj` → 任意读写 → ArrayBuffer backing store 改写 → WASM 代码执行。map 索引、对象偏移和指针压缩布局都绑定给定 commit，必须以题目版本为准。
