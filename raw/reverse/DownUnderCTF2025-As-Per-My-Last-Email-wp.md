# As per my last email...

## 题目简述

题目提供一篇关于 VI 与 Emacs 的长文本 `lock`，以及一串由 Vim 普通模式命令、搜索、寄存器操作和控制键记号组成的 `key`。`key` 实际是一段 Vim 宏：它从长文多个位置拼接 flag，随后故意清空缓冲区制造“结果消失”的假象，但在此之前已把结果保存到命名寄存器 `z`。

## 解题过程

先把 `key` 放进 Vimscript 的寄存器赋值中：

```vim
" solve.vim
let @q = '<macro_goes_here>'
```

附件使用 `<CR>`、`<ESC>`、`<BS>` 表示特殊按键。若直接复制为单引号字符串，这些文字不会自动变成控制键；应参考 `:help key-notation`，在插入模式中用 `Ctrl-V` 后跟相应按键写入真实控制字符。官方仓库里的 `solve/solve.vim` 已保存转换后的宏，可作为准确版本。

用 Vim/Neovim 打开 `lock` 后执行：

```vim
:source solve.vim
@q
```

宏会搜索诸如 `Du`、`modal`、`VI`、`Unix`、`Emac` 等锚点，使用 `"a`、`"b`、`"c`、`"d` 等命名寄存器暂存片段，经过移动、删除和拼接形成 flag。结尾的 `ggdG` 会删除整个缓冲区，所以屏幕上看不到结果；但此前的 `"zy...` 已把目标文本写入 `z` 寄存器。

查看或粘贴该寄存器：

```vim
:registers z
"zp
```

得到：

```text
DUCTF{3Welc0me_.Tru3-VI-cult1s7!}
```

## 方法总结

- 核心技巧：把附件识别为 Vim 宏，执行后追踪命名寄存器，而不是只观察最终缓冲区。
- 识别信号：`gg`、`/pattern<CR>`、`yi{`、`"ay`、`p` 等组合明显属于 Vim 普通模式命令，题面又围绕编辑器之争。
- 复用要点：宏中的特殊键必须按 Vim key notation 正确编码；当宏故意删除或改乱缓冲区时，用 `:registers` 检查命名寄存器和删除寄存器，它们常保留中间结果。
