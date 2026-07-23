# Detective Pikachu 1

## 题目简述

这是一道多层证据恢复题：预期附件是加密的 `sinnoh_dossier.zip`，内部包含
`crime_scene.jpg` 和 `case_logs.wav`，两个媒体文件中又用 Steghide 嵌入了提示、二次加密档案及虚拟机镜像。

需要特别说明：官方仓库把本题列在“上传文件错误或不可解”的题目中，公开目录未保留可复现附件。以下过程是依据仓库中的官方开发者解题记录和已记录的口令重建的预期解法，不把缺失附件伪装成已复测结果。

## 解题过程

### 打开外层档案并提取媒体载荷

开发者记录说明，外层 ZIP 口令位于常见弱口令字典中：

```text
DNALOCS=NEVER
```

解压后分别检查 JPEG 和 WAV。两者都使用 Steghide，而不是把 ZIP 简单追加在文件尾部：

```bash
steghide info crime_scene.jpg
steghide extract -sf crime_scene.jpg

steghide info case_logs.wav
steghide extract -sf case_logs.wav
```

`crime_scene.jpg` 中预期得到 `lol.txt` 和 `sticky_note.txt`。前者说明原有城镇名单已被销毁；后者的“Secret Stuff”部分使用单字节
$0x19$ 异或，解密后提示需要按全国图鉴构造宝可梦候选词表。

### 构造二次 ZIP 口令

`case_logs.wav` 中预期得到 `case_files.zip`。其口令不是宝可梦名称本身，而是对带图鉴编号的候选字符串重复进行 500 次 BLAKE2b：

```python
import hashlib

def derive(name, dex_number):
    value = f"{name}-#{dex_number:04d}".encode()
    for _ in range(500):
        value = hashlib.blake2b(value).hexdigest().encode()
    return value.decode()
```

对完整图鉴候选枚举后，命中的是：

```text
Klawf-#0950
```

其 500 轮 BLAKE2b 十六进制结果为：

```text
a714a5860c272b74adae64489d92855ef88b6c1610975480018d30a34a9b10c00ca8e09ae088623002bc8eaa58b0aefe596ffe120bb3a6111999b7f862da4ca0
```

用该值解开 `case_files.zip`，预期得到虚拟机镜像和 `flag1.txt`：

```text
UMDCTF{d373c71v3_p1k4chu_h07_0n_7h3_c453}
```

虚拟机镜像是下一题的输入，不应在本题继续猜测其中内容。

## 方法总结

- 核心技巧：沿“加密 ZIP → JPEG/WAV Steghide → XOR 提示 → 图鉴词表 → 迭代 BLAKE2b 口令”的证据链逐层恢复。
- 识别信号：多种看似正常的媒体和档案同时出现时，应记录每层输入、输出、口令来源和哈希变换，避免把载荷层级混在一起。
- 复用要点：迭代哈希使用的是每轮 `hexdigest()` 的 ASCII 结果，而非原始 digest 字节；候选格式中的大小写、`-#` 和四位补零都必须精确一致。
