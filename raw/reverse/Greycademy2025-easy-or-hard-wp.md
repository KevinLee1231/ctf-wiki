# Easy or Hard

## 题目简述

分发文件是一个静态 ELF：外层带反调试和时间检查，用一份故意改坏常量的 ChaCha20 解密内嵌 ELF，并通过 `memfd_create`、`fexecve` 无落地执行。内层程序再用标准 ChaCha20 解密请求地址。地址路径本身就是 flag，因此无需依赖已下线或已变更的赛时服务。

## 解题过程

外层先读取 `/proc/self/status` 检查 `TracerPid`，并测量解密前后的耗时；超过约 10 毫秒便退出。真正载荷存放在全局 `buf`，用固定 key、nonce 解密后写入匿名 memfd：

```c
chacha20_init_context(&ctx, key, nonce, 0);
chacha20_xor(&ctx, buf, PL_LEN);
int fd = memfd_create("easy_or_hard", MFD_CLOEXEC);
write(fd, buf, PL_LEN);
fexecve(fd, NULL, NULL);
```

外层算法与标准 ChaCha20 只有一处关键差异：常量从 `expand 32-byte k` 改成了 `expand 32 byte k`。逆向分发二进制时必须保留这个差异才能正确恢复内层 ELF；直接套标准库会得到错误数据。

内层 `http_get` 使用标准 ChaCha20。提取其 32 字节 key、12 字节 nonce 和 63 字节密文后可直接解密：

```python
from Crypto.Cipher import ChaCha20

key = bytes.fromhex(
    "8f2ad561e87b01b2b860088806da6f060046d9624c6bf8daedc82ba5c232487f"
)
nonce = bytes.fromhex("4025e247e0c24f0d9856daaf")
ciphertext = bytes.fromhex(
    "b2becb5fa39f6589ae6c20c588ad61dd7ed22f6290eeed8b72973dcc09c8fbd2"
    "961bfed7a33a2d232316df19ef01aee7e7c198ce343606f7bd89b5a2bae03c"
)

print(ChaCha20.new(key=key, nonce=nonce).decrypt(ciphertext).decode())
```

明文是一个 HTTP 地址，其中路径为：

```text
grey{h0P3_Y0u_DiD_IT_Th3_345Y_w4Y}
```

当前域名已不再提供赛时内容，但这不影响复现，因为 flag 完整存在于本地密文的解密结果中。

## 方法总结

面对内存加载器，应把问题拆成“恢复第一层载荷”和“分析第二层业务逻辑”。密码算法名称相同不代表实现相同：一个字符的常量变化就会改变全部密钥流。最终结果若已编码在请求地址中，应以本地解密为证据，不把易失的远程响应当作必要步骤。
