# BYUCTF 2023 - Ducky2

## 题目简述

第二个 Rubber Ducky 载荷不再使用 US 布局，而是按 Slovak `SK.json` 编译。使用默认映射可以大致读出字母，但 flag 尾部符号会全部错误。

## 解题过程

仍按 8 字节 HID 报告拆分，但把 keycode、普通字符和 Shift 字符映射替换为 Hak5 的 Slovak 布局表。错误的 US 解码能给出类似 `makesurezourkezboardissetupright` 的提示，正是在说明 `y/z` 和符号键布局不匹配。

用 `SK.json` 重新解码后，完整文本为：

```text
byuctf{makesureyourkeyboardissetupright)@&%(#@)!(#*$)}
```

## 方法总结

“文本大部分可读”并不证明布局正确。区域键盘经常只交换少数字母，却彻底改变数字行的上档符号；flag 含大量标点时，必须以题面语言线索选择准确布局。
