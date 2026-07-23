# UMDCTF 2025 - talk-tuah

## 题目简述

程序允许用户提交一个话题和一段文本，并把文本保存到随机目录
`/tmp/talk_tuah/<24 位随机串>/<topic>`。再次提交同名话题时，程序会读取先前的文件。
容器中的目标是读取 `/app/flag.txt`。

虽然题目最初放在 `misc`，但决定性障碍是利用文件检查与打开之间的竞争窗口，
因此归入 `pwn`。

## 解题过程

阅读 `src/main.rs` 可以看到，`remember_thought` 依次执行了三件事：

```rust
let m = fs::metadata(&p)?;
if file_uid != my_uid && file_gid != my_gid {
    return Err("wrong owner".into());
}

let mut f = File::open(&p)?;
if f.metadata()?.is_symlink() {
    return Err("symlink".into());
}
```

这里有两个关键问题：

1. `fs::metadata(&p)` 会跟随符号链接，并且所有权检查与 `File::open` 之间没有原子性；
2. `f.metadata()` 检查的是已经打开的目标文件描述符。即使路径原本是符号链接，
   它返回的也是目标文件元数据，因此 `is_symlink()` 不会发现链接本身。

主程序启动时会直接打印随机存储目录，所以不需要猜路径。先创建普通话题 `hi`，
使 `<随机目录>/hi` 成为当前用户拥有的普通文件；随后再次提交 `hi`，触发
`remember_thought`。在第一次 `metadata` 检查通过后、`File::open` 执行前，
将该文件替换为指向 `/app/flag.txt` 的符号链接：

```rust
fs::remove_file(&topic_path)?;
std::os::unix::fs::symlink("/app/flag.txt", &topic_path)?;
```

竞争时序可概括为：

```text
服务：metadata(普通文件) ──────────────── File::open(同一路径)
利用：                 unlink -> symlink("/app/flag.txt")
```

窗口很短，所以官方解题程序不断启动新进程并重复竞争。成功时，所有权检查看到的是
普通文件，而 `File::open` 打开的已经是 flag；后续 `f.metadata().is_symlink()`
检查目标文件，只会得到 `false`。最终读到：

```text
UMDCTF{more_tocttou_bugs_next_week_with_hailey_welch}
```

## 方法总结

本题是典型的 TOCTOU 路径竞争。防御时不能把“检查路径”和“按路径打开”分成两个
可被替换的操作；应使用 `openat` 一类接口配合 `O_NOFOLLOW`，并在受控目录描述符下
打开文件，再对最终文件描述符验证所有权。仅在 `open` 之后调用
`f.metadata().is_symlink()` 并不能检测原路径是否为符号链接。
