# ezWin 7zip

## 题目简述

第三道 ezWin 子题仍使用同一份 Windows 10 内存镜像。目标是定位并导出桌面上的 `flag.7z`，再根据压缩包内文件名提示恢复当前用户密码，解开归档并取得 flag。

## 解题过程

### 定位并导出归档

使用文件对象扫描插件查找 `flag.7z`：

```sh
python3 vol.py \
  -f win10_22h2_19045.2486.vmem \
  windows.filescan.FileScan | grep -i 'flag\.7z'
```

镜像中出现两个同名文件对象：

```text
0xd0064181c950  \Users\Noname\Desktop\flag.7z
0xd00641b5ba70  \Users\Noname\Desktop\flag.7z
```

若一个对象的 `DataSectionObject` 导出失败，应继续尝试另一个对象或其 `SharedCacheMap`，而不是认定文件已经损坏。官方复现使用第二个对象：

```sh
python3 vol.py \
  -f win10_22h2_19045.2486.vmem \
  windows.dumpfiles.DumpFiles \
  --virtaddr 0xd00641b5ba70
```

生成的 `.dat` 文件经 `file` 识别为 7-Zip archive，直接改名或交给 `7z` 即可。归档中只有一个文件：

```text
crack_nt_hash_for_7z_pwd.txt
```

### 从 NT hash 恢复压缩密码

文件名提示压缩密码就是当前用户 NT hash 对应的明文。与 `ezWin auth` 相同，运行：

```sh
python3 vol.py \
  -f win10_22h2_19045.2486.vmem \
  windows.hashdump.Hashdump
```

得到 `Noname` 用户的 NT hash：

```text
84b0d9c9f830238933e7131d60ac6436
```

这是常见弱口令的 NTLM 值，可用字典离线恢复，避免依赖在线解密站点：

```sh
printf '%s\n' '84b0d9c9f830238933e7131d60ac6436' > ntlm.txt
hashcat -m 1000 ntlm.txt rockyou.txt
hashcat -m 1000 ntlm.txt --show
```

明文为：

```text
asdqwe123
```

用它解压导出的归档：

```sh
7z x flag.7z -pasdqwe123
```

文本内容为：

```text
hgame{e30b6984-615c-4d26-b0c4-f455fa7202e2}
```

## 方法总结

内存中的同一路径可能对应多个 `_FILE_OBJECT`，实际数据也可能只保留在共享缓存映射中。`dumpfiles` 失败时应遍历同名对象并检查输出类型。取得加密归档后，文件名往往也是取证线索；本题需要把文件扫描、账户哈希提取、弱口令恢复和归档解压串成一条完整证据链。
