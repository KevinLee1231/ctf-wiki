# Reject to Inject

## 题目简述

附件是 64 位 Windows DLL `IV.dll`。`DllMain` 在 `DLL_PROCESS_ATTACH` 时创建线程执行 `MAIN`；该线程不会校验用户输入，而是通过 WinAPI 获取当前用户目录和宿主可执行文件路径，只有路径完全符合预期时才解码内置字符串并弹窗显示 flag。

关键点是 `GetModuleFileNameA(0, ...)` 的第一个参数为 `NULL`。它返回当前进程主可执行文件的完整路径，而不是 `IV.dll` 自身的位置。因此题目名称虽然暗示 DLL injection，实际不需要向其他进程注入代码，只需要构造满足路径检查的正常 DLL 宿主。

## 解题过程

### 恢复路径检查

DLL 的核心逻辑可整理为：

```cpp
GetUserProfileDirectory(token, user_profile, &size);

strcpy_s(expected, user_profile);
strcat_s(expected, "\\Room2004");
strcat_s(expected, "\\sigpwnie.exe");

GetModuleFileNameA(NULL, actual, sizeof(actual));

if (strncmp(expected, actual, sizeof(actual)) == 0) {
    decode(encoded, output);
    MessageBoxA(NULL, (char *)output, "Success", 0);
}
```

假设用户目录为 `C:\Users\alice`，则必须让宿主进程路径为：

```text
C:\Users\alice\Room2004\sigpwnie.exe
```

DLL 位置没有参与比较，但用相对路径加载时，把 `IV.dll` 与 EXE 放在同一目录最简单。

### 编写同架构加载器

创建 `sigpwnie.cpp`：

```cpp
#include <Windows.h>
#include <stdio.h>

int main() {
    HMODULE module = LoadLibraryA("IV.dll");
    if (module == NULL) {
        printf("LoadLibraryA failed: %lu\n", GetLastError());
        return 1;
    }

    // DLL_PROCESS_ATTACH 中创建的工作线程需要时间执行并显示 MessageBox。
    puts("Close the flag dialog, then press Enter...");
    getchar();
    FreeLibrary(module);
    return 0;
}
```

使用 x64 Visual Studio Developer PowerShell 编译：

```powershell
cl /nologo /EHsc /Fe:sigpwnie.exe sigpwnie.cpp
```

随后创建目录并放置文件：

```text
%USERPROFILE%\Room2004\sigpwnie.exe
%USERPROFILE%\Room2004\IV.dll
```

从该目录启动 `sigpwnie.exe`。`LoadLibraryA` 触发 DLL attach，工作线程获取到的用户目录与主模块路径拼接后完全一致，于是进入 `decode()`。解码器先以一个假 flag 字符串初始化 PRNG，恢复自定义 Base32 字母表，再解码固定字符串 `IS7WX...`；无需手工逆完这部分，因为满足路径条件后 DLL 会直接完成解码并在消息框中显示：

```text
uiuctf{0uch_righ7_0n_7h3_sp07}
```

如果弹窗没有出现，应依次检查：EXE 是否确切命名为 `sigpwnie.exe`、目录大小写和层级是否一致、DLL 与加载器是否同为 x64，以及主进程是否在工作线程完成前退出。

## 方法总结

- 核心技巧：识别 `GetModuleFileNameA(NULL, ...)` 检查的是 DLL 宿主 EXE，把加载器命名并放置到 `%USERPROFILE%\Room2004\sigpwnie.exe` 后正常加载 DLL。
- 识别信号：DLL attach 中出现 `GetUserProfileDirectory`、字符串拼接、`GetModuleFileNameA` 和路径比较，说明环境条件比内置编码算法更接近真正的门禁。
- 复用要点：逆向 Windows 模块路径时必须区分 `NULL`、EXE 模块句柄和 DLL 自身 `HMODULE` 的语义；还要匹配位数，并避免宿主进程过早退出导致 attach 中创建的线程来不及执行。
