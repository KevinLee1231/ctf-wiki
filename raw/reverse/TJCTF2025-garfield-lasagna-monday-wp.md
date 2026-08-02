# garfield-lasagna-monday

## 题目简述

网页让用户填写 16 个 Mad Lib 单词，但真正的校验只发生在浏览器下载的 `challenge.wasm` 中。前端把前五项用 `|` 连接后传给导出函数 `check`；返回 1 时再调用 `get_flag`，从 WebAssembly 线性内存读取 flag。决定性障碍是恢复 WASM 的比较常量，而不是攻击 Flask 后端。

## 解题过程

`mylabs.html` 中的 JavaScript 明确给出了数据流：

```javascript
const checkInput = `${w1}|${w2}|${w3}|${w4}|${w5}`;
u8.set(new TextEncoder().encode(checkInput), 1024);
if (exports.check(1024) === 1) {
    const flagPtr = exports.get_flag();
    // 从 flagPtr 读取到 NUL
}
```

下载 `/static/challenge.wasm` 后可用 `wasm2wat challenge.wasm -o challenge.wat` 转成文本。`check` 对偏移 `0..31` 逐字节执行 `i32.load8_u`，紧随其后的 `i32.const` 就是期望字符。按偏移排序并转成 ASCII，得到：

```text
blue|tuxedo|dance|chaos|pancakes
```

因此前五个输入依次填写 `blue`、`tuxedo`、`dance`、`chaos`、`pancakes`。其余 11 项不参与校验，可任意填写。

编译后的 `get_flag` 更直接：它把三个 64 位常量按小端序写入地址 `1024`、`1032`、`1040`，随后补 NUL。无需运行网页即可解码：

```python
parts = [
    3708568498132445812,
    7596551555448135522,
    9038006696380691298,
]
flag = b"".join(x.to_bytes(8, "little") for x in parts).decode()
print(flag)
```

结果为：

```text
tjctf{w3b_m4d_libs_w4sm}
```

仓库中的 C 源码展示了编译前的另一层伪装：输入常量与 `0x55` 异或，flag 则用固定初始状态 `0x13371337` 的 LCG 字节流异或；但编译后的 WAT 已把比较字节和解密结果具体化，直接分析实际下发的 WASM 更短也更贴近客户端执行逻辑。

## 方法总结

- 核心技巧：顺着前端 JavaScript 找到 WASM 导出函数，再从 WAT 的逐字节比较和小端内存写入中恢复输入与 flag。
- 识别信号：关键判断完全在客户端执行、静态目录公开 `.wasm`、页面脚本直接调用 `instance.exports`。
- 复用要点：WASM 的 `i64.store` 按线性内存的小端字节序落盘；分析时还要区分服务端模板逻辑、前端胶水代码和真正的校验模块。
