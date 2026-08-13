# GreyCTF2024 Greycat's Adventure - Achievement 3 WP

## 题目简述

题目提示“秘密藏在制作人员名单中”，并要求把秘密输入文本框。附件仍是 Greycat's Adventure 的 Unity IL2CPP 构建；关键不是逐帧观看名单，而是检查游戏元数据中的明文字符串并把与 credits 提示相符的 flag 单独对应到本题。

## 解题过程

对完整游戏目录做递归二进制文本搜索：

```bash
grep -aR -n "grey{" "GreyCat's Adventure_Data"
```

搜索会定位到 `global-metadata.dat`。进一步运行：

```bash
strings "GreyCat's Adventure_Data/il2cpp_data/Metadata/global-metadata.dat" \
  | grep -F "grey{"
```

候选项中，描述“猫已经逃出袋子”的字符串与隐藏秘密这一题意相符：

```text
grey{th3_c4t_15_0u7_0f_th3_b4g_la138df}
```

将该字符串输入题目所说的文本框并让输入框失去焦点，即可让游戏执行校验并触发成就。这里的界面操作只是验证，答案本身已经从元数据完整恢复。

## 方法总结

制作人员名单、对话和 UI 文本常被 Unity 构建收进全局元数据。遇到“隐藏在 credits 中”的提示，应优先搜索元数据和资源字符串；若同一文件出现多个 flag，再根据挑战叙事和触发行为完成映射，而不是随意选择第一个结果。
