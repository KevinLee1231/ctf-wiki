# TSGCTF2024 Cached File Viewer

## 题目简述

程序允许把文件加载到 10 个槽位，并用 `std::string_view` 引用缓存内容。若文件名包含 `flag`，首次加载时会把对应槽位标记为 `is_redacted=true`，读取只能看到 `**redacted**`。

赛中第一版的缓存命中分支如下：

```cpp
if (arena.find(filename) != arena.end()) {
    items[index].str = arena[filename];
    items[index].is_redacted = false;
    return;
}
```

该分支复用了缓存字符串，却没有重新按文件名设置敏感标记。

## 解题过程

先把 `flag` 加载到任意槽位，例如槽位 1：

```text
choice > 1
index > 1
filename > flag
Read 22 bytes.
```

`update_items` 检测到文件名含 `flag`，所以槽位 1 被标记为 redacted；与此同时，文件内容已经保存到全局 `arena["flag"]`。

再次加载同一个文件，可以使用另一个槽位，也可以覆盖原槽位：

```text
choice > 1
index > 2
filename > flag
```

这次命中 `arena`，程序直接让 `items[2].str` 指向缓存中的真实 flag，却无条件把 `items[2].is_redacted` 设为 `false`。读取槽位 2：

```text
choice > 2
index > 2
content: TSGCTF{!7esuVVz2n@!Fm}
```

因此 flag 为：

```text
TSGCTF{!7esuVVz2n@!Fm}
```

第一版还存在后续修正版题目利用的 `string_view` 生命周期问题，但本题不需要触发内存 UAF；缓存标记不一致已经构成最短、稳定的泄露路径。

## 方法总结

本题是缓存对象与访问控制元数据脱节：首次加载路径正确设置 redaction，缓存命中路径却把相同敏感内容标记为公开。安全属性必须附着在被缓存对象本身，不能由每个引用槽位自行、重复且不一致地推导。修复时缓存命中分支至少应使用 `filename.find("flag") != npos`，更稳妥的是让 `arena` 同时保存内容与敏感级别，并禁止调用者覆盖该级别。
