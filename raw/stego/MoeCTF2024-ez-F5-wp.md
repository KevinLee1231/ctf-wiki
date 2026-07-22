# ez_F5

## 题目简述

附件是一张使用 F5 算法嵌入数据的 JPEG。密码没有直接写在正文，而是先以 Base32 放进图片备注；正确理解“未提供密码”和“密码错误”两类提取结果，是本题比普通工具题多出的判断点。

## 解题过程

查看图片属性，在“备注”字段中得到：

```text
NZXV64DBONZXO33SMQ======
```

Base32 解码结果为：

```text
no_password
```

F5 并不是在像素最低位直接写数据，而是修改 JPEG 量化后的非零 DCT 系数，并用密码派生的伪随机排列决定读取顺序。因而同一张图片在密码缺失或错误时仍可能“提取出东西”，但长度和内容不可信。

使用 [F5 参考实现](https://github.com/matthewgao/F5-steganography)时，`Extract` 的调用形式是 `java Extract <jpeg> -p <password>`，默认把隐藏文件恢复为嵌入时记录的文件名。本题先故意省略密码：

```bash
java Extract suantouwangba.jpg
```

工具把错误排列解释成极大的嵌入长度，关键输出为：

```text
Length of embedded file: 8337781 bytes
Incomplete file: only 0 of 8337781 bytes extracted
```

输入错误密码时，工具可能产出一段随机字节，而不是明确报“密码错误”。截图中的 `output.txt` 长度为 2432 字节，内容是高熵乱码，没有合理文本结构。

使用元数据恢复出的密码重新运行：

```bash
java Extract suantouwangba.jpg -p no_password
```

此时 `output.txt` 中得到：

```text
moectf{F5_15_s0_lntere5t1n9}
```

## 方法总结

判断 F5 提取是否成功不能只看进程退出码：异常的嵌入长度、只写出 0 字节或高熵乱码都说明密码/排列不对。元数据中的 Base32 是密码线索而不是“无需密码”的字面提示；恢复后还应检查输出是否具有合理长度与 flag 结构。
