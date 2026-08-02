# TSGCTF2024 Cached File Viewer 2

## 题目简述

修正版把缓存命中逻辑改为：

```cpp
items[index].is_redacted = filename.find("flag") != std::string::npos;
```

因此重复加载 flag 已不能直接清除敏感标记。剩余漏洞在 `update_items` 对 `std::string_view` 的生命周期管理：

```cpp
std::string_view str = *content;
/* 可能先输出旧 items[index] */
items[index].str = str;
arena[filename] = std::move(*content);
```

`items[index].str` 借用了临时 `std::string` 的字符区，但程序先建立 view，再移动字符串并销毁原对象；标准并不保证这个 view 仍有效。

## 解题过程

### 1. 利用 Small String Optimization 制造悬空 view

对于较长字符串，移动构造通常会把堆缓冲区所有权转给 `arena`，旧 view 在实现层面往往仍碰巧指向有效缓冲区。题目使用的 libc++ 对短字符串采用 Small String Optimization（SSO）：短于 23 字节的内容直接存放在 `std::string` 对象内部，不另行分配字符缓冲区。

加载一个 22 字节普通文件：

```text
/var/lib/dpkg/info/libdb5.3t64:amd64.shlibs
```

其内容被放在 `unique_ptr<std::string>` 指向对象的内联区。`string_view` 指向这块对象内存；移动到 `arena` 时字符被复制到目标对象自己的 SSO 区，原对象随后释放，所以槽位中的 view 悬空。此时槽位的 `is_redacted=false`。

### 2. 用同长度 flag 复用临时字符串对象

接着在同一槽位加载 22 字节的 `flag`。`file_reader` 再次通过 `make_unique<std::string>` 创建临时字符串，分配器会复用刚释放的对象地址；flag 字符因此正好写入旧 view 指向的 SSO 内联区。

`update_items` 在更新槽位和设置 redaction 之前先处理已有内容：

```cpp
if (!items[index].str.empty()) {
    output_content(items[index]);
    std::cout << "Overwrite loaded file? (y/n) > ";
    /* ... */
}
```

旧 `string_view` 现在读到新临时对象中的 flag，而旧敏感标记仍为 `false`，于是程序在询问是否覆盖之前直接输出：

```text
content: TSGCTF{hQAz-yXc6fLoyK}
```

完整交互顺序为：

```text
load_file(index=1, filename=/var/lib/dpkg/info/libdb5.3t64:amd64.shlibs)
load_file(index=1, filename=flag)
```

第二次操作输出旧槽位内容时即可截获 flag，无需回答后续覆盖提示。

## 方法总结

本题是所有权移动与非拥有 view 混用导致的 UAF，SSO 使问题从“通常碰巧可用”变为可稳定复用的对象内联缓冲区。`std::string_view` 不延长底层字符串寿命；建立 view 后再移动或销毁源对象，不能依赖实现细节保持指针有效。修复方法是先把内容放入 `arena`，再令 view 指向 `arena` 中生命周期稳定的字符串，或让 `Item` 直接拥有 `std::string`；敏感标记也应与内容一起存储。
