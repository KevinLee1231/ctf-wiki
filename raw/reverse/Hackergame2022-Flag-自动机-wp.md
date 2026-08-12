# Flag 自动机

## 题目简述

附件是一个 Win32 GUI 程序。鼠标靠近“狠心夺取”按钮时，按钮会随机移动；即使偶然点中，正常点击也不会写出 flag。需要逆向窗口消息处理逻辑，构造程序真正认可的消息。

## 解题过程

### 定位窗口过程

在 IDA 或 Ghidra 中分析 `WinMain`。程序用 `RegisterClassW` 注册窗口类，把窗口过程设置为一个内部函数，随后由 `CreateWindowExW` 创建主窗口并进入标准消息循环。继续查看窗口过程，可以看到与 flag 有关的分支：

```c
case 0x111:
    if ((WORD)wParam == 3 && lParam == 114514) {
        plaintext = decrypt_flag(hwnd, 114514);
        fp = fopen("flag_machine.txt", "w");
        fwrite(plaintext, 1, strlen(plaintext), fp);
        fclose(fp);
    }
    break;
```

`0x111` 是 `WM_COMMAND`。按钮的普通点击确实会生成该消息，但其 `lParam` 不会是常量 `114514`，所以直接点击只会进入失败路径。

### 主动发送正确消息

窗口类名和标题均为 UTF-16LE 字符串 `flag 自动机`。可以再编译一个很小的 Win32 程序，先取得主窗口句柄，再发送逆向得到的参数：

```c
#include <windows.h>
#include <stdio.h>

int main(void) {
    HWND target = FindWindowW(L"flag 自动机", L"flag 自动机");
    if (target == NULL) {
        puts("window not found");
        return 1;
    }

    if (!PostMessageW(target, 0x111, 3, 114514)) {
        puts("PostMessageW failed");
        return 1;
    }
    return 0;
}
```

先启动题目程序，再运行辅助程序。正确消息经过消息队列送入目标窗口过程，解密逻辑随即执行，并在当前目录生成 `flag_machine.txt`。

### 二进制补丁方案

若不编写辅助程序，也可以修改两个条件跳转：

1. `0x401966` 处把 `jnz` 的操作码 `0F 85` 改为 `jz` 的 `0F 84`，让按钮不再随鼠标移动；
2. `0x401811` 处把 `jz` 的操作码 `74` 改为 `jnz` 的 `75`，绕过 `lParam == 114514` 的限制。

保存补丁后的程序，再正常点击按钮，也会触发文件写出逻辑。相比继续静态还原带花指令的完整解密算法，伪造消息或修改条件跳转都更直接，因为它们复用了程序自身的解密函数。

## 方法总结

Win32 GUI 程序通常围绕“消息循环—窗口过程”组织逻辑。逆向这类程序时，不应只跟踪可见控件，还应重点检查 `WndProc` 中的 `Msg`、`wParam`、`lParam` 条件。只要找到隐藏功能真正要求的消息参数，就能通过 `FindWindowW` 与 `PostMessageW` 重放事件，绕开界面层面的干扰。
