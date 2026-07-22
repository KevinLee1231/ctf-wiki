# BlackHole

## 题目简述

主程序通过 `LoadLibraryA` 加载 VMProtect 保护的 `you_cannot_crack_me.vmp.dll`，再调用导出函数 `checkMyFlag(char *, size_t)`。题目纸条已经给出 7 个内部字符的位置约束，因此无需先脱掉 VMP；把 DLL 当作黑盒校验 oracle，枚举约 176 万个候选即可。

## 解题过程

纸条给出的内部七字符约束是：

- 第 1 位为 `c`；
- 第 3、7 位是数字；
- 第 6 位为 `m`；
- 第 2、4、5 位是小写字母。

候选数为 $26^3\times10^2=1,757,600$。动态加载 DLL 后获取导出函数地址，固定已知位置并枚举其余五位：

```cpp
#include <cstdio>
#include <windows.h>

using CheckFlag = int (*)(char *, size_t);

int main() {
    HMODULE module = LoadLibraryA("you_cannot_crack_me.vmp.dll");
    if (module == nullptr) {
        std::fprintf(stderr, "LoadLibrary failed: %lu\n", GetLastError());
        return 1;
    }

    auto check = reinterpret_cast<CheckFlag>(
        GetProcAddress(module, "checkMyFlag")
    );
    if (check == nullptr) {
        std::fprintf(stderr, "export not found\n");
        FreeLibrary(module);
        return 1;
    }

    char flag[16] = "moectf{c????m?}";
    for (char p2 = 'a'; p2 <= 'z'; ++p2)
    for (char p3 = '0'; p3 <= '9'; ++p3)
    for (char p4 = 'a'; p4 <= 'z'; ++p4)
    for (char p5 = 'a'; p5 <= 'z'; ++p5)
    for (char p7 = '0'; p7 <= '9'; ++p7) {
        flag[8] = p2;
        flag[9] = p3;
        flag[10] = p4;
        flag[11] = p5;
        flag[13] = p7;
        if (check(flag, 15)) {
            std::printf("%s\n", flag);
            FreeLibrary(module);
            return 0;
        }
    }

    FreeLibrary(module);
    return 1;
}
```

运行得到：

```text
moectf{cr4ckm3}
```

原稿在模块句柄上调用 `CloseHandle`，这是错误的资源释放 API；`LoadLibrary` 获得的 `HMODULE` 应由 `FreeLibrary` 释放。

## 方法总结

重保护并不意味着必须先还原全部内部逻辑。只要目标仍导出确定性的校验函数，就能把它当作 oracle；题面约束把搜索空间降到百万量级后，直接原生调用比对 VMP 虚拟机做完整逆向更可靠。
