# SHmAc

## 题目简述

服务签发如下形式的访问令牌：

```text
Base64(|user:<name>|admin:0|expires:<timestamp>|).Base64(mac)
```

登录时先校验 MAC，再从左到右解析 `user`、`admin` 和 `expires` 字段；管理员令牌可以读取 flag。所谓 `SHmAc` 并不是标准 HMAC，而是把 32 字节密钥直接写入 SHA-256 的八个内部状态字，然后对令牌正文执行 `SHA256_Update` 和 `SHA256_Final`。

这个设计会把最终摘要直接暴露为可继续计算的 SHA-256 链式状态，因此可以进行长度扩展。令牌解析器还允许字段重复，后出现的 `admin` 和 `expires` 会覆盖前面的值。

## 解题过程

注册一个较短的用户名后，可以得到合法的普通用户令牌。设解码后的正文为 $m$，MAC 解码后的 32 字节摘要为 $h$。源码中的自定义初始化只替换 SHA-256 状态，并没有把密钥字节计入消息长度，所以原消息的 SHA-256 glue padding 完全可由攻击者计算：

```python
from struct import pack


def sha256_padding(message_length):
    return (
        b"\x80"
        + b"\x00" * ((55 - message_length) % 64)
        + pack(">Q", message_length * 8)
    )
```

选择较短用户名时，$m$ 加 padding 后正好占一个 64 字节块。把摘要 $h$ 的八个大端 32 位字作为新的 SHA-256 状态，并把“已经处理的长度”设为 64 字节，就能继续哈希攻击者追加的字段：

```text
|admin:1|expires:<未来时间戳>|
```

最终伪造内容为：

```text
m || glue_padding(m) || |admin:1|expires:<未来时间戳>|
```

官方 solver 通过一个 OpenSSL C 辅助程序完成“从指定摘要状态继续 SHA-256”：先让 `SHA256_CTX` 计入 64 字节已处理长度，再用原 MAC 覆盖 `ctx.h[0..7]`，最后更新追加数据并调用 `SHA256_Final`。使用支持自定义起始摘要的长度扩展库时，等价的令牌构造可以写成下面的 Python 骨架：

```python
from base64 import b64decode, b64encode
import time

import hlextend


def forge(access_token: bytes) -> bytes:
    data_b64, mac_b64 = access_token.split(b".", 1)
    original = b64decode(data_b64)
    digest = b64decode(mac_b64)
    suffix = f"|admin:1|expires:{int(time.time()) + 3600}|".encode()

    # SHmAc 的密钥只作为初始状态，不计入消息长度，因此 secretLength=0。
    extender = hlextend.new("sha256")
    forged_text = extender.extend(
        suffix.decode("ascii"),
        original.decode("ascii"),
        0,
        digest.hex(),
    )
    forged_body = (
        forged_text.encode("latin-1")
        if isinstance(forged_text, str)
        else bytes(forged_text)
    )
    forged_mac = bytes.fromhex(extender.hexdigest())
    return b64encode(forged_body) + b"." + b64encode(forged_mac)
```

将伪造令牌提交给登录功能后，MAC 校验仍然通过。解析器先读到原来的 `admin:0`，随后跨过二进制 padding，又读到追加的 `admin:1` 和未来过期时间；后写入的值覆盖旧值，服务最终输出：

```text
shellmates{go77a_st1ck_7o_s7andard5_th3n_:P}
```

## 方法总结

本题同时利用了两个设计错误：自制 keyed hash 暴露了可继续计算的 SHA-256 状态，以及令牌语法允许安全属性重复且采用后值覆盖。仅有长度扩展只能追加数据，若解析器坚持首字段优先或拒绝重复字段，仍不能把 `admin:0` 改为 `admin:1`。

看到“直接修改哈希初始状态”“`hash(key || message)` 替代 HMAC”或摘要可被当作内部状态恢复时，应检查长度扩展；同时必须审计消息解析语义、padding 后字节如何被跳过、字段冲突规则和已处理长度。标准 HMAC 会在内外两层哈希中隔离内部状态，应优先使用成熟实现而不是自制 MAC。
