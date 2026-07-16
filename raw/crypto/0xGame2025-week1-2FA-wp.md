# 2FA

## 题目简述

题目实现了一个基于 TOTP 的双因素认证服务。注册时，服务端随机生成 20 字节密钥，将其编码为 Base32，并通过 `otpauth://` URI 生成二维码；登录时校验用户提交的一次性验证码，验证成功后必须在 10 秒内请求 flag。

## 解题过程

源码中的关键流程如下：

```python
key = os.urandom(20)
secret = b32encode(key).decode()
totp = pyotp.TOTP(secret)
uri = totp.provisioning_uri(name=username, issuer_name="0xGame2025")

if pyotp.TOTP(secret).verify(token):
    logged = True
    logged_time = time.time()
```

注册后，用任意支持 TOTP 的 Authenticator 扫描终端输出的二维码。Authenticator 与服务端持有同一个 Base32 密钥，并根据当前时间片计算 6 位验证码，因此无需攻击随机数或绕过校验。提交当前验证码完成登录，再立即选择获取 flag，确保没有超过源码规定的 10 秒窗口。

如果二维码在 Windows 终端中显示乱码，可先执行 `chcp 65001` 将代码页切换为 UTF-8；这只是终端显示问题，不影响 TOTP 算法。

## 方法总结

- 核心技巧：识别并正常完成标准 TOTP 注册、验证流程。
- 识别信号：Base32 密钥、`otpauth://` URI、`pyotp.TOTP` 和动态 6 位验证码共同指向基于时间的一次性密码。
- 复用要点：先确认共享密钥和时间同步，再关注登录态是否存在额外的有效期限制；本题的关键限制是验证后 10 秒内取 flag。
