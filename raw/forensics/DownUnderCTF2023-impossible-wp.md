# DownUnderCTF 2023 impossible Writeup

## 题目简述

题目发布了一个 Flask 应用归档。公开的 `crypto.py` 含有一组故意设置的诱饵密钥、IV 和密文，并声称线上实例已经更换全部值。真正的部署常量残留在误打包的 `app/utils/__pycache__/crypto.cpython-310.pyc` 中。

## 解题过程

先列出归档内容，会发现源码旁边保留了 Python 3.10 字节码缓存。`.pyc` 的代码对象会保存字符串常量，即使没有反编译器，直接提取可打印字符串也能看到部署时的 AES 参数：

```bash
strings app/utils/__pycache__/crypto.cpython-310.pyc
```

输出中的关键内容为：

```text
ed4e0cc3a8d5d267bc4f1924c552676291a20c681acd8d97c6cdb4b091c705b375e104714d69647541957f82b70cc54705f47c03a5a3b7e95fcb0eb8097d2c0b209c9e60508c0379500c8bb94ad588540bb11c75bff4b44887398b608e3323e17fb3f31b3c8a7a46cae69563014962cc92440c92021d79b17f12e329a371a97f
f122df4b445b2c383ace204f1571e410d7c5061c8852ed0b1f1a5e696aab0bea
b9e3fb697dba55f8753921b88acb8509
```

三项依次是 `FLAG_ENC`、`KEY` 和 `IV`。原题解中的 `stringsftw.png` 只是上述终端输出截图，没有额外视觉证据，因此这里将内容转写为文本，不再保留图片。

线上 Flask 路由把查询参数 `key` 交给 `decrypt`，且会先与真正的 `KEY` 字符串比较。提交：

```text
f122df4b445b2c383ace204f1571e410d7c5061c8852ed0b1f1a5e696aab0bea
```

服务使用 AES-CBC 解密并显示：

```text
DUCTF{o0p5_i_f0rG0t_aBoUt_pYc4Ch3!!1!}
```

## 方法总结

这不是破解 PBKDF2 或 AES，而是发布物取证。构建归档时遗留的 `__pycache__` 仍含源码常量，而且它对应线上真实配置，优先级高于旁边的诱饵源码。发布前应从干净构建上下文生成制品，并排除字节码缓存、调试文件和历史构建产物。
