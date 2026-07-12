# SUCTF2026-easyLLM

## 题目简述
题目把 LLM 输出作为 AES 密钥材料。服务端每次返回一段 JSON，包含 `AES-128-CBC`、`iv_b64`、`ciphertext_b64`、密钥派生规则 `key = SHA256(LLM_output)[:16]`，以及完整的 LLM 调用参数：provider、模型 `GLM-4-Flash`、temperature、system prompt 和 user prompt。

关键机制是 LLM 低温采样的输出空间并不等同于强随机密钥空间。由于模型和 prompt 已知、temperature 较低，可以用相同参数大量采样候选输出，再把候选输出哈希成 AES key 去碰撞解密多组靶机密文。

## 解题过程
访问靶机（三个端口均返回相同结构），得到一段 JSON：

```
{
"algo": "AES-128-CBC",
"iv_b64": "CTo1mJkt5TAvjUqoS/n+uQ==",
"ciphertext_b64":
"jBhtdnA6jfGpq0yzXWQsRJlLvRd6nFL6xefha2MDglFjSdTBl3CQe5IxIUNh84Ny",
"key_derivation": "key = SHA256(LLM_output)[:16]",
"llm": {
"provider": "z.ai",
"model": "GLM-4-Flash",
"temperature": 0.28,
"system_prompt": "You are a password generator.\nOutput ONE password
only.\nFormat strictly: pw-xxxxxxxx where x are letters.\nNo explanation, no
quotes, no punctuation.",
"user_prompt": "Generate the password now."
}
}
```

给出了加密方式和 LLM 的调用参数，需要还原 LLM 输出以推导 AES 密钥并解密密文

```
LLM_output = GLM-4-Flash(system_prompt, user_prompt, temperature=0.28)
key = SHA256(LLM_output.encode("utf-8"))[:16] # 取前16字节作为AES-128密钥
ciphertext = AES-128-CBC(plaintext, key, iv)
```

从返回的参数可知：

1. LLM 参数：模型、温度、prompt 全部已知，可以用相同参数调用同一个模型

2. temperature=0.28：较低的温度意味着输出分布集中，候选空间有限

3. 每次访问靶机都会重新生成：三个端口每次请求返回不同的 iv 和 ciphertext，说明服务端每次都重
新调用 LLM 生成新密码并加密

由于 temperature=0.28 较低，LLM 输出空间有限（实测约 60-70 种不同输出），而靶机每次访问都
重新生成密码。因此：

1. 从靶机大量收集密文：每组密文对应一个不同的 LLM 输出

2. 用相同参数大量采样 LLM：收集足够多的候选密码

3. 交叉碰撞：用所有候选密码逐一尝试解密所有密文，只要某个采样恰好命中某组密文的密码，即可
解密得到 Flag

最后脚本如下

```python
#!/usr/bin/env python3

import hashlib
import json
import requests
from base64 import b64decode
from collections import Counter
from Crypto.Cipher import AES
from zhipuai import ZhipuAI

API_KEY = "AI API Key"
TARGETS = [
"http://<target>:10013",
"http://<target>:10014",
"http://<target>:10015",
]
SYSTEM_PROMPT = (
"You are a password generator.\n"
"Output ONE password only.\n"
"Format strictly: pw-xxxxxxxx where x are letters.\n"
"No explanation, no quotes, no punctuation."
)
USER_PROMPT = "Generate the password now."
N_CHALLENGES = 10
N_LLM_SAMPLES = 100

def try_decrypt(password: str, iv: bytes, ciphertext: bytes) -> str | None:
key = hashlib.sha256(password.encode()).digest()[:16]
cipher = AES.new(key, AES.MODE_CBC, iv)
pt = cipher.decrypt(ciphertext)
pad = pt[-1]
if 0 < pad <= 16 and all(b == pad for b in pt[-pad:]):
try:
result = pt[:-pad].decode("utf-8")
if result.isprintable():
return result
except UnicodeDecodeError:
pass
return None

def collect_challenges() -> list[dict]:
challenges = []
for url in TARGETS:
for _ in range(N_CHALLENGES):
try:
r = requests.get(url, timeout=5)

data = r.json()
challenges.append({
"iv": b64decode(data["iv_b64"]),
"ct": b64decode(data["ciphertext_b64"]),
})
except Exception as e:
print(f" 请求失败 {url}: {e}")
return challenges

def collect_llm_outputs(client: ZhipuAI) -> list[str]:
outputs = []
for i in range(N_LLM_SAMPLES):
try:
response = client.chat.completions.create(
model="glm-4-flash",
temperature=0.28,
messages=[
{"role": "system", "content": SYSTEM_PROMPT},
{"role": "user", "content": USER_PROMPT},
],
)
outputs.append(response.choices[0].message.content)
except Exception as e:
print(f" LLM 调用失败: {e}")
if (i + 1) % 20 == 0:
print(f" 采样进度: {i+1}/{N_LLM_SAMPLES}")
return outputs

def make_variants(raw_outputs: list[str]) -> set[str]:
candidates = set()
for output in raw_outputs:
candidates.add(output)
candidates.add(output.strip())
candidates.add(output.rstrip('\n'))
stripped = output.strip()
candidates.add(stripped + "\n")
return candidates

def main():
client = ZhipuAI(api_key=API_KEY)

print(f"[*] Step 1: 从靶机收集密文 ({N_CHALLENGES}次 x {len(TARGETS)}端
口)...")
challenges = collect_challenges()

print(f" 共收集 {len(challenges)} 组密文\n")

print(f"[*] Step 2: 采样 GLM-4-Flash ({N_LLM_SAMPLES}次)...")
raw_outputs = collect_llm_outputs(client)
candidates = make_variants(raw_outputs)

counter = Counter([o.strip() for o in raw_outputs])
print(f" 唯一输出: {len(counter)} 种")
print(" Top 5:")
for pw, cnt in counter.most_common(5):
print(f" {pw:25s} x{cnt}")

total = len(candidates) * len(challenges)
print(f"\n[*] Step 3: 交叉碰撞 ({len(candidates)} 候选 x {len(challenges)} 密
文 = {total} 次)...")

for pw in candidates:
for ch in challenges:
result = try_decrypt(pw, ch["iv"], ch["ct"])
if result:
print(f"\n{'='*50}")
print(f"[+] 解密成功!")
print(f" Password : {repr(pw)}")
print(f" Flag : {result}")
print(f"{'='*50}")
return

print("\n[-] 未命中，建议增大 N_CHALLENGES 和 N_LLM_SAMPLES 后重试")

if __name__ == "__main__":
main()
```

运行结果如下

```
[*] Step 1: 从靶机收集密文 (10次 x 3端口)...
共收集 30 组密文

[*] Step 2: 采样 GLM-4-Flash (100次)...
采样进度: 20/100
采样进度: 40/100
采样进度: 60/100
采样进度: 80/100
采样进度: 100/100

唯一输出: 65 种
Top 5:
pw-AbcDfghIjkl x9
pw-Abcde1f x9
pw-9z8v7b6 x7
pw-AbcDfgh x6
pw-AbcdeFg x3

[*] Step 3: 交叉碰撞 (148 候选 x 30 密文 = 4440 次)...

==================================================
[+] 解密成功!
Password : 'pw-9v2k8p6z'
Flag : SUCTF{LLM_w1ll_ch4nge_ev3rything}
==================================================
```

Crypto

## 方法总结
- 核心技巧：LLM 低温采样候选碰撞
- 识别信号：加密密钥依赖 LLM 输出，且 prompt/模型/temperature 已知。
- 复用要点：大量采集靶机密文，同时本地按相同参数采样候选输出；用 AES 解密校验交叉碰撞，找到正确输出。
