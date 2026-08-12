# Fort Knockies

## 题目简述

题目给出的是一个本地密码管理器的保存容器镜像。最终文件系统看起来已清理敏感文件，但容器镜像是分层交付：后续层的删除只写入 whiteout 标记，不会从较早 layer 的 `layer.tar` 擦除数据。解题目标是从旧层恢复开发遗留物，重建旧格式加密文件的口令与算法，再解密 flag。

关键工件来自三个已删除路径：`/app/.env`、`/app/.git/` 和 `/var/lib/fortknockies/.staging/README`。这属于对既有镜像层和项目历史的证据恢复，而不是对密码原语本身的攻击，因此归类为 Forensics。

## 解题过程

### 从旧容器层恢复被删除文件

先将题目提供的 image archive 导入并重新导出为可检查的 OCI/Docker tar，再按 manifest 的 layer 顺序解开每个 `layer.tar`。不要只查看最终容器：在最终层中看到白化条目只能说明路径曾被删除，实际内容要到删除之前的层找。

```sh
docker load -i <challenge-image.tar>
docker image save <loaded-image> -o image.tar
tar -xf image.tar
# 逐个检查 manifest.json 所列目录中的 layer.tar
```

在旧层恢复如下材料：

- `.env` 的末尾孤立一行 `pycache`；
- Git 历史中的旧 `scratch_crypto.py` 包含 `part2 = "PATH"`；
- `.staging/README` 不是文本。其前六字节为 `37 7a bc af 27 1c`，即 7z 文件签名。

因此口令不是猜测，而是两份独立遗留物拼接的 `pycachePATH`。将伪装的 README 改作 7z 数据解压后，得到 `sample-upload.enc` 与 `flag.enc`；前者使用当前 `FKENC1` 格式，是用于误导的样本，目标是旧格式的 `flag.enc`。

### 复刻 FKENC0 并解密

旧 Git 脚本记录了已废弃的 `FKENC0` 导入格式：salt、IV、ciphertext 均为 Base64；从口令派生 32 字节 key 使用 PBKDF2-HMAC-SHA1、64000 次迭代；密文是 AES-256-CBC 加 PKCS7 padding。等价解密函数如下：

```python
import base64, json
from pathlib import Path
from cryptography.hazmat.primitives import hashes, padding
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC

def b64d(value: str) -> bytes:
    return base64.b64decode(value)

def decrypt_fkenc0(path: Path, password: str) -> bytes:
    blob = json.loads(path.read_text(encoding="utf-8"))
    if blob.get("version") != "FKENC0":
        raise ValueError("expected FKENC0")
    key = PBKDF2HMAC(
        algorithm=hashes.SHA1(), length=32,
        salt=b64d(blob["salt_b64"]),
        iterations=int(blob["iterations"]),
    ).derive(password.encode())
    decryptor = Cipher(
        algorithms.AES(key), modes.CBC(b64d(blob["iv_b64"]))
    ).decryptor()
    padded = decryptor.update(b64d(blob["ciphertext_b64"])) + decryptor.finalize()
    unpadder = padding.PKCS7(128).unpadder()
    return unpadder.update(padded) + unpadder.finalize()

print(decrypt_fkenc0(Path("flag.enc"), "pycachePATH").decode().strip())
```

对题目附带的 `solve/flag.enc` 运行官方 solver 已实际验证，输出为：

```text
grey{jz_some_rookie_mistakesi9v2k}
```

## 方法总结

- 核心技巧：把容器镜像当作分层证据源，利用旧 layer、whiteout 和 Git 历史恢复已删除配置、脚本和伪装归档。
- 识别信号：题目声称文件已清理却给出容器镜像；路径名像 README 但文件签名不是文本；同一应用同时存在新旧版本的私有封装格式。
- 复用要点：先保存 layer 顺序和删除前后的证据链，再根据 magic bytes 而非文件名判断真实格式。解密前核对 version、KDF、迭代次数、key 长度、cipher mode 与 padding；当前格式的样本文件不应替代从历史中恢复的旧格式定义。
