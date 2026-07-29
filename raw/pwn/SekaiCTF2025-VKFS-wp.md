# VKFS

## 题目简述

VKFS 是一个把文件块映射到 Vulkan sparse image 坐标的 FUSE 文件系统。路径先做 SHA-256，再被压缩为：

```c
coord.mip     = hash_qword0 % 6 + 1;
coord.block_x = hash_qword1 & max_val;
coord.block_y = hash_qword2 & max_val;
```

最终 inode 只有 16 位：

```text
mmmm xxxxxx yyyyyy
```

大量不同路径会映射到相同坐标。目录项又采用无长度字段的 `[u16 inode][NUL 结尾文件名]`，增删改名时依赖零字节扫描与 `strlen`。官方解法组合可控哈希碰撞、目录项重排和大文件后继块元数据伪造，最终从保存 flag 的 Vulkan 块读取数据。

## 解题过程

### 1. 生成三类特殊文件名

官方 `generate_payload.py` 不要求完整 SHA-256 碰撞，只要求哈希映射后的坐标或 inode 满足目标条件，因此暴力成本很低。

生成器分别寻找：

1. `payload0`：其完整长路径与 `/A.../B.../z` 映射到同一 Vulkan 坐标；
2. `payload1`：长度 6，映射到 inode `0x1000`；
3. `payload2`：首字节为 `0x70`，映射到 inode `0x2000`。

核心判断只比较：

```python
hash0 % 6 + 1
hash1 & mask
hash2 & mask
```

因此名称可同时承载目录项伪造字节和指定 inode。

### 2. 利用目录项布局

`place_dirent()` 查找空位时，先从 `curr+2` 开始统计连续零字节，再依赖：

```c
curr += sizeof(uint16_t) + strlen(filename) + 1;
```

它没有独立保存每条目录项长度。删除、硬链接和重命名后，攻击者控制的文件名字节可以被重新解释为：

- 下一条目录项的 inode；
- 文件名终止符；
- Vulkan 文件头中的字段；
- 大文件后继块编号。

官方 `exp.c` 先建立两层接近路径长度上限的 `A.../B...` 目录及 `z` 子目录，再把普通节点重命名为 `payload0`。该名称与原 `z` 路径映射到同一坐标，使两套目录操作落到同一底层块并重排其字节。

### 3. 伪造大文件链

VKFS 文件头记录 `next_lfs_block`。读取超过单块容量时，文件系统按该值下降一个 mip，并选择四个子块之一：

```c
coord->mip--;
coord->block_x = coord->block_x * 2 + (block & 1);
coord->block_y = coord->block_y * 2 + ((block & 2) >> 1);
```

官方利用再通过 `payload1` 的 `0x1000` inode、硬链接和第二次 rename，把目录数据调整成伪造文件头及后继块指针。随后创建和删除若干短名称，令 `payload2` 对应的目录项以预期边界出现。

这里不能只把它概括成普通路径哈希碰撞：碰撞用于让两个目录视图共享块，真正越权读取依赖目录项字节被解释为大文件元数据。

### 4. 读取 flag 块

最终打开的是：

```c
open(PATH_BASE + "/" + (payload2 + 1), O_RDONLY);
```

即跳过 `payload2` 的首个 `0x70` 控制字节，把剩余部分当作可见文件名。随后执行：

```c
pread(fd, buf, 0x100, 0xffa8);
```

加上文件头大小后，读取位置跨过 `0x10000` 边界，触发伪造的 `next_lfs_block` 跟随逻辑，最终把坐标切换到保存 flag 的底层块并返回其内容。

仓库正式挑战镜像中的 flag 为：

```text
SEKAI{f1l35y573m5_1n_gpU_m3m0ry_4r3_7h3_fu7ur3_7ru57_m3}
```

## 方法总结

本题的安全边界同时依赖“路径到 16 位坐标的压缩”和“无长度目录项解析”，两者都不能承受攻击者可控输入：

- 16 位 inode 必然产生大量可搜索碰撞；
- 文件名不能兼任结构边界；
- rename/link/unlink 后必须保持目录结构一致性；
- 大文件链指针必须验证目标块归属和循环。

哈希并不会自动提供唯一性。若哈希只截断到很小空间，就必须用完整路径或强唯一标识解决冲突，并为所有磁盘结构使用显式长度、校验和与范围检查。
