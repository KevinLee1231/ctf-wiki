# SUCTF2026-easyLLM

## 题目简述

题目把一次大语言模型输出直接用作 AES 密钥材料。服务返回：

- 算法：AES-128-CBC + PKCS#7 padding；
- `iv_b64` 和 `ciphertext_b64`；
- 派生规则：`SHA256(LLM_output)[:16]`；
- LLM provider、模型名、temperature、system prompt 和 user prompt。

system prompt 要求模型只输出形如 `pw-xxxxxxxx` 的密码，temperature 为 `0.28`。低温并不使输出确定，但会让概率质量集中在一批高频字符串上。攻击者可以用完全相同的模型参数反复采样候选输出，对每个候选派生 AES key，再尝试解密题目密文。

源码中的密码、key、IV 和密文都在 FastAPI 模块加载时生成一次并保存为全局变量；同一未重启实例的多次 GET 请求返回相同 JSON。比赛时提供了多个独立实例，因此可得到多组分别在启动时采样的密文，从而提高候选与任一目标相撞的概率。不能把它误写成“每次 HTTP 请求都会重新调用 LLM”。

## 解题过程

### 还原服务端加密流程

服务启动时执行的关键逻辑是：

```python
password = call_llm_once(
    model="GLM-4-Flash",
    temperature=0.28,
    system_prompt=SYSTEM_PROMPT,
    user_prompt="Generate the password now.",
).strip()

key = hashlib.sha256(password.encode("utf-8")).digest()[:16]
iv = secrets.token_bytes(16)
ciphertext = AES_CBC_PKCS7_encrypt(key, iv, FLAG.encode())
```

需要精确复现以下细节：

- 模型名、两段 prompt、temperature 必须相同；
- 服务端对模型回复执行了 `.strip()`，哈希的是去掉首尾空白后的 UTF-8 字节；
- SHA-256 摘要取前 16 字节，不是取十六进制字符串前 16 个字符；
- CBC 解密后必须做严格的 PKCS#7 unpad。

官方 WP 中的调用脚本曾直接写入 API 凭据。长期 WP 不应保存真实 token；应从环境变量读取，并通过当前 provider 的兼容接口发起请求：

```python
API_KEY = os.environ["LLM_API_KEY"]

def sample_password(client):
    response = client.chat.completions.create(
        model="glm-4-flash",
        temperature=0.28,
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": "Generate the password now."},
        ],
    )
    return response.choices[0].message.content.strip()
```

模型版本和服务端采样实现可能随时间变化；若赛后使用同名滚动模型，输出分布未必仍与比赛时一致。这是该解法的外部状态依赖，不应把比赛期命中率写成永久保证。

### 扩大目标集合与候选集合

对每个独立题目实例读取一组 `(iv, ciphertext)`。同一实例重复读取没有收益；只有不同进程、不同实例或重新启动后的实例才可能对应不同 LLM 密码。

随后使用相同调用参数采样大量候选。总 PDF 中的比赛期实测为 100 次采样得到约 65 种去重输出，说明分布明显集中，但这只是当时模型版本的观测值。

攻击成功概率取决于候选集合与目标实例集合的笛卡尔积。若有 $r$ 个候选密码和 $t$ 组独立密文，最多只需进行 $r\times t$ 次本地 AES 解密，不必为每次组合再次调用 LLM。

### 严格验证解密结果

候选 key 的尝试逻辑如下：

```python
def try_decrypt(password: str, iv: bytes, ciphertext: bytes) -> str | None:
    key = hashlib.sha256(password.encode("utf-8")).digest()[:16]
    cipher = AES.new(key, AES.MODE_CBC, iv)
    padded = cipher.decrypt(ciphertext)

    pad = padded[-1]
    if not (1 <= pad <= 16 and padded[-pad:] == bytes([pad]) * pad):
        return None

    try:
        plaintext = padded[:-pad].decode("utf-8")
    except UnicodeDecodeError:
        return None
    return plaintext if plaintext.startswith("SUCTF{") and plaintext.endswith("}") else None
```

只检查 padding 或“可打印文本”不够严格：错误 key 偶然产生合法 padding 的概率并非 0。结合已知 flag 前后缀验证，可以显著降低误报。

完整流程骨架为：

```python
targets = load_independent_instances()  # [(iv, ciphertext), ...]
candidates = {sample_password(client) for _ in range(sample_count)}

for password in candidates:
    for iv, ciphertext in targets:
        plaintext = try_decrypt(password, iv, ciphertext)
        if plaintext is not None:
            print("password:", repr(password))
            print("flag:", plaintext)
            raise SystemExit
```

如果没有命中，应增加独立实例数或 LLM 采样数，并统计高频候选确认采样参数是否正确；不应不断重复抓取同一进程返回的同一密文。若当前 SDK 会保留尾部换行，而服务端源码仍执行 `.strip()`，候选侧也必须做相同规范化，不需要额外枚举带换行版本。

比赛期参赛队通过多实例密文与候选采样交叉碰撞，命中密码 `pw-9v2k8p6z`，解得：

```text
SUCTF{LLM_w1ll_ch4nge_ev3rything}
```

## 方法总结

- 核心技巧：对低温 LLM 的集中输出分布做采样字典攻击，再按精确派生规则碰撞 AES-CBC 密钥。
- 识别信号：密钥直接来自 LLM 输出，模型、prompt、temperature 和规范化规则全部公开，且能取得一个或多个独立密文样本。
- 复用要点：复刻模型调用和字符串规范化的每个字节细节；区分“进程启动时采样一次”和“每个请求采样一次”。用严格 padding、UTF-8 和 flag 格式联合验证，API 凭据只从环境变量读取，并意识到滚动模型版本会让赛后复现发生漂移。
