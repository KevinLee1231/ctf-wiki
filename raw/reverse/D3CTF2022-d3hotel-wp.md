# d3hotel

## 题目简述

本题由两部分组成，包括 luac 逆向以及 wasm 逆向。首先下载得到 BuildWebGL.wasm 和

BuildWebGL.data。

整体流程是 Unity WebGL 题：Lua 部分会给出一个假 flag，真正校验在 IL2CPP 生成的 wasm 函数里。需要从 `global-metadata.dat`、`main.lua`、`script.json` 和 `BuildWebGL.framework.js` 之间建立函数编号映射，最终定位 `D3Checker::Check`。

## 解题过程

使用 AssetStudio 打开 BuildWebGL.wasm ，然后点击 File -> Extract file ，提取 global-metadata.dat和 main.lua。使用 unluac 反编译 main.lua，然后通过求解代数问题可以得到假 flag：

d3ctf{W3ba5m_1s_Awe5oom3} 。

使用 Il2CppDumper 解析 BuildWebGL.data，检查生成的 script.json，可以找到 D3Checker::Check的偏移 7581 以及调用类型 vii。

```json
"Address": 7581,
"Name": "D3Checker$$Check",
"Signature": "void D3Checker__Check (D3Checker_o* __this, const MethodInfo* method);",
"TypeSignature": "vii"
```

打开 BuildWebGL.framework.js，可以找到 vii 调用类型对应 wasm 中的导出函数 zh。

```javascript
var dynCall_vii = Module["dynCall_vii"] = function() {
    return (dynCall_vii = Module["dynCall_vii"] = Module["asm"]
["zh"]).apply(null, arguments)
}
```

使用 ghidra-wasm-plugin 插件加载 BuildWebGL.wasm，找到导出函数 zh。

```wat
export::zh
    local.get        l1
    local.get        l2
    local.get        l0
    call_indirect    type= 0x1   table0
    end
```

参考 WebAssembly 文本格式，这里 call_indirect 指令的参数对应 wasmTable 中的序号。

在 js 加载 wasm 文件的地方下断点，然后检查 wasmTable 的 7581 项，可以得到函数在

BuildWebGL.wasm 文件中的编号为 39846。找到函数 unnamed_function_39846，发现

D3Checker::Check 对 flag 的第 10 位进行了替换，替换后可以得到真 flag：

d3ctf{W3b@5m_1s_Awe5oom3} 。

```javascript
wasmTable.get(7581)
ƒ $func39846() { [native code] }
```

这里给出另外一种解法，检查生成的 script.json，可以找到字符串 Congratulation  的地址为

0x24fac4。启用 Scalar Operand References 分析，或者在 Ghidra 中搜索这个地址，可以找到来自函数 unnamed_function_39846 的引用，从而定位到 D3Checker::Check 的位置。

```json
"Address": 2423492,
"Value": "Congratulation"
```

## 方法总结

- 核心技巧：Unity WebGL 题要把 metadata 中的函数地址、JS glue code 的 `dynCall_*` 类型和 wasm table 下标对应起来。
- 定位路径：先用 Lua 反编译排除假 flag，再用 Il2CppDumper 找 `D3Checker::Check` 的地址和签名，最后通过 wasm table 定位到真实 wasm 函数。
- 复用要点：除了按函数编号追踪，也可以从成功提示字符串 `Congratulation` 的地址出发，在 Ghidra 中启用 Scalar Operand References 后反向定位引用函数。
