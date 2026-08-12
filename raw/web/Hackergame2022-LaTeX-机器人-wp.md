# LaTeX 机器人

## 题目简述

服务端把用户输入原样插入 `base.tex` 的 `$$ ... $$` 之间，再以低权限用户运行 `pdflatex -no-shell-escape`，最后把 PDF 转成图片。虽然禁用了 shell escape，但 TeX 本身仍有文件读取能力。目标是读取根目录的 `/flag1` 和 `/flag2`；前者只含字母数字，后者还包含 `_` 与 `#`，会触发 TeX 的特殊字符语义。

## 解题过程

### 确认注入位置

后端脚本的实际拼接过程是：

```bash
head -n 3 /app/base.tex > /dev/shm/result.tex
cat /dev/shm/input.tex >> /dev/shm/result.tex
tail -n 2 /app/base.tex >> /dev/shm/result.tex
pdflatex -interaction=nonstopmode -halt-on-error -no-shell-escape result.tex
```

基础模板为：

```tex
\documentclass[preview]{standalone}
\begin{document}
$$
$$
\end{document}
```

因此用户输入会被 TeX 解释执行，而不是作为普通公式文本转义。

### 读取纯文本 flag

TeX 的 `\input` 可以直接把文件内容读入当前文档：

```tex
\input{/flag1}
```

花括号会被当作分组符而不显示，但其中只含字母和数字，正文仍能完整读出第一段 flag 的内容，再补回 `flag{...}` 结构即可。`-no-shell-escape` 只禁止外部命令，不会禁止 `\input`。

### 处理 `_` 与 `#`

第二个文件不能直接 `\input`：`_` 在数学模式中表示下标，`#` 是宏参数字符，未经处理会报错或改变显示。官方解法先退出模板已经打开的数学模式，逐行读取文件，再用 `\detokenize` 把特殊 token 转成可打印字符：

```tex
$$
\newread\myread
\openin\myread=/flag2
\read\myread to\fileline
\detokenize\expandafter{\fileline}
$$
```

首尾的 `$$` 分别关闭和重新打开外层数学模式，使模板最终仍能正常闭合。渲染结果中的低横线对应 `_`；读取过程中 `#` 的 token 表现可能成对出现，按 TeX 的参数字符规则还原即可。

另一种直接做法是在读取文件前把两个字符的 catcode 改成普通字符：

```tex
$$
\catcode`\_=12
\catcode`\#=12
\input{/flag2}
$$
```

catcode 修改发生在 `/flag2` 被词法分析之前，所以文件中的 `_` 和 `#` 不再具有下标或参数含义，可以按字面渲染。

## 方法总结

- 核心技巧：利用未转义的 TeX 输入和 `\input`/文件流原语读取服务端文件，再处理特殊字符的 catcode。
- 识别信号：Web 服务把用户输入拼入 TeX 模板并调用完整 TeX 引擎，即使使用 `-no-shell-escape`，也仍可能存在文件读取能力。
- 复用要点：沙箱不能只禁 shell escape；还应隔离文件系统、使用受限引擎并严格转义输入。处理未知文本时，要考虑 TeX 在读取阶段就赋予字符语法类别。
