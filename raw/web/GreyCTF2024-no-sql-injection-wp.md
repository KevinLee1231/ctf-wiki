# no-sql-injection

## 题目简述

注册第一步把 `{"name": username, "admin": false}` 编码成 Base64 token 并存入 MySQL；第二步先用 token 查询数据库，再在 Node.js 中 Base64 解码和 JSON 解析。MySQL 默认字符串排序规则不区分大小写，而 Base64 区分大小写，因此“数据库认为相同”的两个 token 可以解码成完全不同的 JSON。

## 解题过程

危险的信任边界是：

```javascript
const result = await query(
    "select 1 from tokens where token = ?", [token]
);
const { name, admin } = JSON.parse(atob(token));
```

查询虽然参数化，不存在 SQL 注入，但比较语义与后续解码语义不一致。例如同一组四个 Base64 字符只改变大小写，就可能从无害高位字节变成引号等 JSON 结构字符。构造两个字符串 `safe` 与 `malicious`，满足：

```python
safe.lower() == malicious.lower()
base64.b64decode(safe)      # 合法的普通注册 JSON
base64.b64decode(malicious) # 注入 admin:true 的 JSON
```

每三个目标明文字节对应四个 Base64 字符。对这四个位置枚举 $2^4$ 种大小写组合，选择一个解码后不含控制字符、引号和反斜杠的组合作为安全用户名片段；恶意 token 则使用能解出目标 JSON 语法的原组合。这样第一步 `JSON.stringify` 仍产生合法的 safe token。

恶意 JSON 需要做两项修改：先从 `name` 字符串中逃逸并插入 `"admin":true`，再把尾部原有的 `"admin":false` 的键名变成另一个合法但无关的键，否则后出现的同名属性会覆盖 true。官方构造最终可解析为类似：

```json
{"name":"sshsh ","admin":true,"asb":"","OTHER_KEY":false}
```

用精心选择的安全用户名完成第一步，使数据库保存 `safe`；第二步提交大小写等价的 `malicious` 与自选密码。MySQL 找到 safe 记录，Node.js 却解析 malicious 并创建管理员账号。随后登录得到：

```text
grey{fr13nd5h1p_3nd3d_w17h_my5ql}
```

## 方法总结

参数化查询只解决注入，不保证字符串身份判断正确。Base64、哈希、签名和 token 都是大小写敏感的精确字节表示，数据库列应使用 binary collation 或二进制类型比较。更稳妥的设计是服务端保存不可伪造的随机 token，并把权限字段留在服务端，而不是再次信任客户端提交的结构化权限数据。
