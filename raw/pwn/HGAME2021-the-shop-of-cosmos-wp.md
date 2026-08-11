# the_shop_of_cosmos

## 题目简述

商店的交易逻辑接受负数数量，购买负数商品反而增加余额。获得足够余额后，程序提供任意文件读取和带偏移写入能力。读取 `/proc/self/maps` 可以得到各映射基址，写 `/proc/self/mem` 则可直接修改当前进程内存；将 `system` 写入 `__free_hook` 后，下一次 `free("/bin/sh")` 即可获得 shell。

## 解题过程

先在购买操作中提交负数，例如 `-30`，利用符号校验缺失增加余额。随后购买文件读取功能并指定：

```text
/proc/self/maps
```

从输出中找到 libc 映射的起始地址。应按映射行解析基址，而不是复制某次运行中的 `0x7f...` 地址：

```python
libc_base = int(mapping_start, 16)
free_hook = libc_base + libc.sym["__free_hook"]
system = libc_base + libc.sym["system"]
```

再使用文件写入功能打开 `/proc/self/mem`，把写入偏移设为 `__free_hook - 8`，写入 16 字节：

```python
payload = b"/bin/sh\x00" + p64(system)
write_file(
    path="/proc/self/mem",
    offset=free_hook - 8,
    data=payload,
)
```

前 8 字节同时保存在程序处理写入数据的临时缓冲区中，后 8 字节覆盖 `__free_hook`。写入流程结束并释放该临时缓冲区时，hook 把原本的 `free(buffer)` 改成 `system(buffer)`；由于缓冲区以 `/bin/sh\x00` 开头，直接执行 shell。

`/proc/self/mem` 以进程虚拟地址作为文件偏移，内核在允许打开后可写入本进程的映射，不受目标页原有 ELF 段权限的常规限制。官方 WP 还记录了非预期解：定位程序只允许打开特定文件名的比较常量，通过 `/proc/self/mem` 改写只读区里的 `"flag"` 字符串，即可让其他路径通过检查后直接读取 flag。

## 方法总结

本题的完整利用链是负数业务逻辑漏洞、`/proc/self/maps` 地址泄露和 `/proc/self/mem` 任意写。看到可控文件路径与偏移时，应检查 Linux procfs 的进程接口，而不只考虑磁盘文件。修复时既要拒绝负数并防止整数溢出，也要限制可打开路径；容器或沙箱还应收紧 procfs、凭据和系统调用权限。
