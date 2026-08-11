# Prisoner Processor

## 题目简述

该服务把带 HMAC 签名的 JSON 转成 YAML 文件。完整利用链由四个互相依赖的缺陷构成：原型污染绕过签名覆盖范围、Bun 对含 NUL 路径的截断、黑名单路径过滤绕过，以及把 YAML 组织为可执行 TypeScript 后重启进程。

最终目的是覆盖被 Bun 运行的 `/app/src/index.ts` 并在重启时调用带 SUID 的 `getflag` 程序，因此按决定性 HTTP/API 逻辑漏洞归入 Web。

## 解题过程

先从 `/examples` 取得一份服务器已签名的 prisoner JSON 与其 `signature`。签名只计算 `getSignedData(data)` 的自有可枚举属性，而提取函数会把 `signed.__proto__` 写到一个普通对象上。加入如下片段后，`outputPrefix` 落在原型中：

```json
"signed.__proto__": {
  "outputPrefix": "../../proc/self/fd/3\u0000"
}
```

`Object.entries` 不会枚举该原型属性，所以原来的合法签名仍可使用；但代码读取 `signedData.outputPrefix` 时会得到污染值。

服务原本把输出文件名拼为 `PREFIX-随机十六进制.yaml`。当 `PREFIX` 含 NUL 时，题目使用的 Bun 文件路径处理会在 NUL 处截断，随机尾部不再生效。简单黑名单禁止 `app`、`src`、`index` 等字串，却没有禁止 `/proc`：运行中的 Bun 把入口脚本打开在文件描述符 3，`/proc/self/fd/3` 因而等价于 `/app/src/index.ts`，可绕过黑名单。

写入内容来自 YAML 序列化，所以要让 YAML 同时是有效 TypeScript。官方思路是在键和值中构造如下骨架，使首行成为可执行语句，其余 YAML 用块注释包住：

```ts
const a: string = Bun.spawnSync({ cmd: ["bash", "-c", "getflag"] });/*
// 后续序列化出的 YAML 位于注释中
z: hi */
```

将这组键值与有效签名一同提交到 `/convert-to-yaml` 后，入口文件被覆写。再触发题目中 Bun/Hono 的未处理异常使守护进程重启，新的 `index.ts` 被加载并执行 `getflag`。源码给出的 flag 为：

```text
DUCTF{bUnBuNbUNbVN_hOn0_tH15_aPp_i5_d0n3!!!one1!!!!}
```

## 方法总结

签名应覆盖规范化后的完整语义对象，且签名验证对象必须是无原型的字典或严格 schema；不能从带 `signed.` 前缀的任意键“筛选”出受保护字段。文件写入必须使用路径规范化、目录约束与允许列表，不能靠子串黑名单。最后，数据格式转换和自动重启的组合会把“任意文件写”升级为代码执行，部署时应避免让运行时入口可写。
