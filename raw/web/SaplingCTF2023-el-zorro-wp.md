# el-zorro

## 题目简述

fox 参数被 os.path.join("assets", "fox_descs", fox) 拼接；当 fox 是绝对路径时，前缀会被丢弃，形成 LFI。点号被过滤，但 /proc/sys/kernel/random/boot_id 不含点，可读取它。应用又把该文件内容 Base64 编码后去掉末尾两个等号作为 Flask SECRET_KEY，因此攻击者能重建签名密钥并伪造会话。

## 解题过程

以任意用户名登录，把 fox 设置为：

~~~text
/proc/sys/kernel/random/boot_id
~~~

访问 /fox 取得 boot_id，按源码计算：

~~~python
secret = base64.b64encode(boot_id.strip().encode()).decode()[:-2]
~~~

管理员名为 admin 加 10 个小写字母。signin 使用 if supplied_name in ADMIN 并在命中时返回 Invalid name，可从 admin 开始逐字符尝试 a-z，借这个子串 oracle 恢复完整管理员名。

最后用泄漏的 secret 签发 Flask session：

~~~json
{"name":"<完整管理员名>","fox":"/flag_la_volpe"}
~~~

带新 session cookie 访问 /fox，startsWith FLAG 的保护分支因管理员名成立而读取文件：

~~~text
maple{but_n0bo0dy_ev3r_a5ks_how_tHe_fOX_f33ls}
~~~

仓库 README 提到 pickle RCE，但实际应用使用 Flask 客户端签名会话，官方脚本也只是伪造 JSON 会话并触发 LFI；复盘以代码和脚本的真实路径为准。

## 方法总结

os.path.join 遇到绝对后项会抛弃前缀，不能用于安全目录约束。更严重的是把可读系统文件派生为 SECRET_KEY，使一次 LFI 直接升级为会话伪造。修复应使用随机独立密钥，并对 realpath 做固定根目录校验。
