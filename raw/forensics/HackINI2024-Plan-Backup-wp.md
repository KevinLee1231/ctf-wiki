# Plan B-ackup

## 题目简述

题目给出一份 Ansible playbook：它把 `~/Desktop/Backup` 归档，用 `my_secret.txt` 作为 Ansible Vault 密码加密，再强制推送到 Git 仓库。解题链需要依次恢复 Git 历史中的 vault 密码、解密 macOS 备份、解析 `.DS_Store` 残留文件名，并用照片 EXIF 中的登录密码解锁 Keychain 安全笔记。

## 解题过程

### 从 Git 历史恢复 vault 密码

playbook 暴露了备份仓库地址和密码文件名：

```bash
git clone https://github.com/4NG3L-4/Backeup.git
cd Backeup
git log --all --reverse -p -- my_secret.txt
```

虽然工作树中已删除 `my_secret.txt`，Git 对象仍保存历史版本。按提交顺序收集该文件每次新增或修改内容的首字符，再按官方解法所示逆序组合并去掉前导干扰字符，可恢复：

```text
q7St!e@5BAy6jBAy
```

### 解密并检查 macOS 工件

先检查附件层级。本仓库中的 `Backup.tar.gz` 外层是 gzip，解压后才出现 `$ANSIBLE_VAULT;1.1;AES256` 头：

```bash
gzip -dc Backup.tar.gz > Backup.vault
ansible-vault decrypt Backup.vault --output Backup.tar
# 输入 q7St!e@5BAy6jBAy
tar -xf Backup.tar
```

在 `Library/FamilyPics` 的 JPEG 上读取 EXIF Comment：

```bash
exiftool -Comment Library/FamilyPics/*.jpg
```

关键内容是：

```text
johnwatson is my login password
```

继续检查 `.Trash` 等目录的 `.DS_Store`。DSStoreParser 会递归解析源目录，并在 miscellaneous TSV 中输出 `record_filename` 字段；本题这些历史文件名采用 Base64，逐项解码后得到 flag 第一部分：

```bash
python3 DSStoreParser.py -s Library -o dsstore-output
cut -f2 dsstore-output/DS_Store-Miscellaneous_Info_Report-*.tsv \
  | while IFS= read -r value; do printf '%s' "$value" | base64 -d 2>/dev/null; done
```

`.DS_Store` 保存 Finder 的文件名、视图和 Trash “放回位置”等元数据，即使原文件已删除，相关记录仍可能残留。[DSStoreParser 官方说明](https://github.com/nicoleibrahim/DSStoreParser)列出了 `-s`、`-o` 参数和三个 TSV 报告的字段含义。

### 解锁 Keychain 取得第二部分

用 EXIF 中的 `johnwatson` 解锁 `login.keychain-db`：

```bash
python -m chainbreaker --dump-all --password johnwatson login.keychain-db
```

在输出的 secure note `flag part 02` 中取得第二部分。Chainbreaker 可在给定登录密码后导出 Generic Password、Internet Password 和 Secure Note 等记录；完整参数见其[官方仓库](https://github.com/n0fate/chainbreaker)。两部分合并为：

```text
shellmates{NaV1gAT3_tH3_unS33n_WHEre_F1l3$_gAThEr_T0_c0nv3Ne}
```

## 方法总结

- 核心技巧：沿 Git 历史、Ansible Vault、EXIF、`.DS_Store` 与 macOS Keychain 串联多类残留证据。
- 识别信号：自动化脚本泄露备份流程和秘密文件名、仓库有强制推送历史、macOS 备份包含 `.Trash` 与 `Keychains`，都指向版本历史和系统元数据恢复。
- 复用要点：每一步得到的 artifact 都是下一步输入；先确认文件真实层级和 magic，再选择解密或解析工具，避免被误导性扩展名带偏。
