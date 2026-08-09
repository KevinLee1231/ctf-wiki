# Cut and Paste

## 题目简述

服务把用户输入拼成 user:{username},admin:false,passwd:{password}，使用 AES-ECB 加密后发放 token。登录时只解密并按逗号解析字段，没有消息认证。ECB 的每个 16 字节块彼此独立，因此可以从不同合法 token 中剪切区块，拼出 admin:true。

## 解题过程

目标是让一个区块以 admin: 开头，再把另一个区块做成带 PKCS#7 填充的 true。字段前缀 user: 长 5 字节。注册用户名 aaaaaaa、密码 true 时，可把 true 调整到独立区块；注册用户名 aaaa、密码 bbbbbbbbbbb 时，可让前一块恰好结束于 admin:。

先把各 token 按 16 字节切块观察：

~~~python
def blocks(token):
    return [token[i:i + 16] for i in range(0, len(token), 16)]

token_prefix = register("aaaa", "bbbbbbbbbbb")
token_true = register("aaaaaaa", "true")

prefix_blocks = blocks(token_prefix)
true_blocks = blocks(token_true)
forged = prefix_blocks[0] + true_blocks[2]
print(login(forged))
~~~

拼接后，解密文本中的关键字段成为 admin:true。服务没有校验密文整体来源，也没有把用户名、权限与密码绑定在同一个认证标签下，因此伪造 token 被接受，返回：

~~~text
maple{ecb_lets_you_go_snip_snip!}
~~~

具体块号应以实际明文和填充结果为准；稳妥做法是先在本地复刻序列化格式，打印每个 16 字节明文块，再选择对应密文块。

## 方法总结

ECB 不隐藏重复结构，也不能阻止攻击者重排合法密文块。即使换成 CBC，只加密不认证仍然可被位翻转。会被服务端信任的 token 应使用 AEAD 模式，或采用成熟的签名会话格式；权限字段尤其不能只依赖客户端可塑的密文。
