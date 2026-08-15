# Free Ram Installer

## 题目简述

附件 `backup.ab` 是一份 Android 设备备份。用户安装了所谓的“RAM installer”后，DCIM 和 Download 目录中的文件都变成了 `.enc`。解题需要从备份中恢复短信、应用和被加密文件，逆向应用取得密钥，再还原照片中的 flag。

## 解题过程

备份头部显示版本为 5、启用了 zlib 压缩且未加密。除了使用 Android Backup Extractor，也可以直接把压缩负载转换为 tar：

```python
from pathlib import Path
import zlib

with open("backup.ab", "rb") as backup:
    assert backup.readline().rstrip() == b"ANDROID BACKUP"
    version = backup.readline().rstrip()
    compressed = backup.readline().rstrip()
    encryption = backup.readline().rstrip()
    payload = backup.read()

assert version == b"5"
assert compressed == b"1"
assert encryption == b"none"
Path("backup.tar").write_bytes(zlib.decompress(payload))
```

解包后最重要的工件包括：

```text
apps/com.android.providers.telephony/d_f/000000_sms_backup
apps/com.example.ram/a/base.apk
shared/0/DCIM/IMG_20220108_081440.jpg.enc
shared/0/DCIM/IMG_20230303_153305.jpg.enc
shared/0/DCIM/IMG_20230419_171332.jpg.enc
shared/0/Download/app-release.apk.enc
```

短信文件自身又经过 zlib 压缩，解压后是 JSON。三条短信说明攻击者以“增加 RAM”为名投递 APK，受害者安装后发现照片被扣留。备份里已经包含 `base.apk`，因此短信中的外部下载地址不是复现解题所必需的依赖。

用 JADX 或 Apktool 检查 `com.example.ram.MainActivity`，可以确认：

- `encryptAll()` 遍历 DCIM 和 Download 目录；
- `A.getKey()` 从 JNI 库 `libnative-lib.so` 取得密钥；
- `Cipher.getInstance("AES")` 未指定模式和填充，在 Android 上等价于 `AES/ECB/PKCS5Padding`；
- 输出文件名是在原文件名后附加 `.enc`。

可以在运行时挂钩 `SecretKeySpec`，也可以单独调用 JNI 导出函数 `Java_com_example_ram_A_keym`。传入应用资源中的参数后，真实 16 字节密钥为：

```text
itdoesstinkalott
```

随后批量解密备份中的四个 `.enc` 文件：

```python
from pathlib import Path
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b"itdoesstinkalott"

for source in Path("recovered-files").glob("*.enc"):
    encrypted = source.read_bytes()
    plain = AES.new(key, AES.MODE_ECB).decrypt(encrypted)
    plain = unpad(plain, AES.block_size)
    source.with_suffix("").write_bytes(plain)
```

三张照片均能恢复为有效 JPEG，下载目录中的文件恢复为有效 APK。其中一张照片只是终端中显示 flag 的文字画面，没有需要依赖原始像素辨认的额外视觉线索，因此直接转写为：

```text
shellmates{android_&_foren$ics=<3}
```

## 方法总结

本题的主线是 Android 备份取证：先从短信建立投递时间线，再从已安装 APK 确认加密范围和算法，最后从 JNI 层恢复 AES 密钥。外部下载链接、普通生活照片和终端文字截图都不是必要证据；完整复现只依赖备份内的工件。最终 flag 为 `shellmates{android_&_foren$ics=<3}`。
