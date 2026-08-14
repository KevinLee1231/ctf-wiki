# bi0sCTF 2024 - ezv8 revenge

## 题目简述

该题修复了原版 `ezv8` 可直接调用 `load()` 的非预期路径，要求利用仓库提供的 V8 补丁。补丁位于 `NodeProperties::InferMapsUnsafe`：遍历 effect chain 遇到并非 map 来源的 `JSCreate` 时，本应把推断结果标记为不可靠，补丁却注释了这一步。攻击者可在 `JSCreate` 的副作用中改变数组元素类型，使优化代码继续使用过期 map，造成类型混淆和越界数组。

完整链为：Proxy 触发 JIT 错误推断、扩大数组越界范围、构造 `addrof` 与堆任意读写、定位 Wasm 实例的代码指针、把另一实例的入口改到浮点常量中夹带的 shellcode，最终执行 `execve("/bin/sh")`。

## 解题过程

### 触发不可靠 map 误判

先准备带 hole 的 double 数组 `a`，通过多次调用让 TurboFan 优化函数：

```javascript
function f(p) {
    a.push(Reflect.construct(function () {}, arguments, p)
           ? 4.183559238858528e-216 : 0);
}
```

把 Proxy 作为 `Reflect.construct` 的第三个参数 `newTarget`。`JSCallReducer::ReduceReflectConstruct` 会生成 `JSCreate`；Proxy 的 `get` trap 在该节点执行副作用，将 `a[1]` 改成对象并创建后续 victim 数组：

```javascript
let p = new Proxy(Object, {
    get: () => {
        a[1] = {};
        oob_arr = [0.1];
        obj_arr = [{}];
        return Object.prototype;
    }
});
```

正常 V8 应在 `JSCreate` 后把 map 视为不可靠并重新检查。题目补丁令编译器继续相信 `a` 仍是原来的 double elements map，最终把对象与浮点布局混用。官方布局下 `oob_arr.length` 被破坏为 `0x8000`，由此获得相邻堆对象的越界读写。

### 建立地址与堆读写原语

用共享 `ArrayBuffer` 在 `Float64` 和 `BigUint64` 间转换位模式。越界数组先定位一个 double victim 数组和一个 object 数组的 map、elements 字段。临时把 object 数组 map 改成 double 数组 map，读取槽位即可得到压缩对象地址；恢复原 map 后形成 `addrof`。

```javascript
function addrof(obj) {
    obj_array[0] = obj;
    let saved = oob_read32(object_map_slot);
    oob_write32(object_map_slot, double_array_map);
    let address = lower32(ftoi(obj_array[0]));
    oob_write32(object_map_slot, saved);
    return address;
}
```

再篡改 victim double 数组的 elements 压缩指针，使 `victim[0]` 指向目标地址。题目构建使用 tagged pointer，官方脚本在地址上应用 `addr - 8 + 1` 的对象头与 tag 修正。这样可实现 cage 内 64 位 `heap_read64`/`heap_write64`。具体 OOB 下标 `40`、`42`、`434` 只适用于附件构建，应通过调试打印对象确认，而不能当作通用 V8 常量。

### 借 Wasm 跳出 Ubercage

仅有 V8 堆读写仍受到 Ubercage 地址空间限制。创建第一个 Wasm 实例，其函数包含一串精心选择的 `f64.const`；这些 8 字节浮点立即数会原样嵌入生成的机器码，拼起来就是 `/bin/sh` 与 `execve` shellcode。通过 `addrof(wasmInstance)` 和实例内固定字段偏移读取编译代码地址，再加上反汇编确定的函数内偏移，得到 shellcode 的实际入口。

随后创建第二个极小 Wasm 实例，把它保存的 RWX/代码入口指针改成第一个实例中的 shellcode 地址：

```javascript
let code = heap_read64(addrof(wasm1) + wasm_code_offset);
let shellcode = code + shellcode_offset;
heap_write64(addrof(wasm2) + wasm_code_offset, shellcode);
wasm2.exports.main();
```

调用第二个导出函数时，V8 沿被篡改的可信代码指针跳出 cage 并执行原生 shellcode。获得 shell 后读取 flag 文件即可。实例字段偏移和 shellcode 落点都必须针对题目 `d8` 版本用调试构建校准。

## 方法总结

漏洞的根因是优化器把带副作用的 `JSCreate` 错当成不会破坏 map 可靠性。Proxy 提供可控副作用，过期 map 造成元素类型混淆，继而扩展为 OOB、`addrof` 和任意堆读写。Ubercage 之后还需单独处理：利用 Wasm 代码对象持有的外部执行指针，并用浮点立即数把 shellcode 放进已编译代码区，最终劫持另一实例的调用入口。
