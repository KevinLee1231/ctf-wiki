# i-heart-wasm

## 题目简述

页面加载一个由 Rust 和 `wasm-bindgen` 生成的 WebAssembly 音频振荡器。音频逻辑只是官方示例改造出的障眼法，flag 没有参与任何导出函数计算，而是逐字符藏在 WASM 的自定义段中。

题目的决定性障碍是检查 WebAssembly 二进制结构，因此归入逆向方向。

## 解题过程

在开发者工具的 Network 面板中找到 `pkg/wasm_test_bg.wasm`。WASM 允许存在不影响执行的 custom section，每个段都有名称和任意字节内容。检查模块可发现名称为字符串 `"0"` 到 `"42"` 的 43 个自定义段，每段内容恰好是一个字符。

段名的顺序与 flag 相反，需要从 42 递减到 0。浏览器原生 API 可以直接完成提取：

```javascript
WebAssembly.compileStreaming(fetch("./pkg/wasm_test_bg.wasm"))
  .then((mod) => {
    const decoder = new TextDecoder();
    let flag = "";

    for (let i = 42; i >= 0; i--) {
      const sections = WebAssembly.Module.customSections(mod, i.toString());
      flag += decoder.decode(sections[0]);
    }

    console.log(flag);
  });
```

输出为：

```text
UMDCTF{4ll_y0ur_53c710n5_4r3_b3l0n6_70_u5}
```

## 方法总结

- WASM 的代码段、数据段和自定义段都应检查，不能只看导出函数。
- `WebAssembly.Module.customSections()` 按段名取内容，返回值是 `ArrayBuffer` 数组，需要再用 `TextDecoder` 解码。
- 音频振荡器与 flag 无关；发现静态资源采用现成 demo 时，应优先比较二进制元数据和附加段。
- 段名本身给出了排序索引，但本题要求逆序拼接。
