# D3lgfs

## 题目简述

题目仿照 Windows Common Log File System（CLFS）漏洞设计了一个内核驱动。原总 WP 把参考漏洞写成了 `CVE-2023-37969`，但所附分析文章和真实编号均为 **CVE-2022-37969**；这里予以纠正。

题目驱动的直接漏洞位于 `addLogContainer`：复制 `containerName` 时没有检查长度，导致固定长度的 `WCHAR containerName[100]` 可向后溢出并覆盖紧邻的 `pContainer` 指针。把该指针改为伪造的 `myContainer` 对象后，`closeLogFile` 会通过伪对象的虚表连续调用方法，从而把普通内存破坏转化为内核控制流劫持。

## 解题过程

### 安装并定位题目驱动

出题环境使用 Windows“添加硬件向导”`hdwwiz.exe` 安装驱动。安装完成后，用户态利用程序需要根据题目提供的设备接口 GUID 枚举设备路径，再用 `CreateFile` 打开设备。

下面的函数返回由 `LocalAlloc` 分配的设备路径副本，调用者使用后应执行 `LocalFree`。与原代码相比，它不会在返回前遗失 `HDEVINFO`，也不会返回一个无法管理生命周期的内部指针：

```c
#include <windows.h>
#include <setupapi.h>
#include <string.h>

#pragma comment(lib, "setupapi.lib")

PCHAR GetDevicePath(const GUID *interfaceGuid)
{
    HDEVINFO info = SetupDiGetClassDevsA(
        interfaceGuid,
        NULL,
        NULL,
        DIGCF_PRESENT | DIGCF_DEVICEINTERFACE
    );
    if (info == INVALID_HANDLE_VALUE) {
        return NULL;
    }

    SP_DEVICE_INTERFACE_DATA interfaceData = {0};
    interfaceData.cbSize = sizeof(SP_DEVICE_INTERFACE_DATA);

    if (!SetupDiEnumDeviceInterfaces(
            info, NULL, interfaceGuid, 0, &interfaceData)) {
        SetupDiDestroyDeviceInfoList(info);
        return NULL;
    }

    DWORD required = 0;
    SetupDiGetDeviceInterfaceDetailA(
        info, &interfaceData, NULL, 0, &required, NULL
    );
    if (required == 0) {
        SetupDiDestroyDeviceInfoList(info);
        return NULL;
    }

    PSP_DEVICE_INTERFACE_DETAIL_DATA_A detail =
        (PSP_DEVICE_INTERFACE_DETAIL_DATA_A)
        LocalAlloc(LMEM_FIXED, required);
    if (detail == NULL) {
        SetupDiDestroyDeviceInfoList(info);
        return NULL;
    }

    detail->cbSize = sizeof(SP_DEVICE_INTERFACE_DETAIL_DATA_A);
    if (!SetupDiGetDeviceInterfaceDetailA(
            info,
            &interfaceData,
            detail,
            required,
            NULL,
            NULL)) {
        LocalFree(detail);
        SetupDiDestroyDeviceInfoList(info);
        return NULL;
    }

    size_t pathLength = strlen(detail->DevicePath) + 1;
    PCHAR path = (PCHAR)LocalAlloc(LMEM_FIXED, pathLength);
    if (path != NULL) {
        memcpy(path, detail->DevicePath, pathLength);
    }

    LocalFree(detail);
    SetupDiDestroyDeviceInfoList(info);
    return path;
}
```

设备接口 GUID 和 IOCTL 编号没有出现在原总 WP 中，不能凭空补写；复现时应从驱动 INF、符号或反编译结果中提取。

### 覆盖 `pContainer`

漏洞对象布局为：

```c
typedef struct my_CLFS_CONTAINER_CONTEXT {
    ULONGLONG cidContainer;
    WCHAR containerName[100];
    union {
        myContainer *pContainer;
        ULONGLONG ullAlignment;
    };
    ULONG cbPrevOffset;
    ULONG cbNextOffset;
} myCLFS_CONTAINER_CONTEXT, *myPCLFS_CONTAINER_CONTEXT;
```

在默认 Windows x64 对齐下，`containerName` 占 $100\times2=200$ 字节；它前面有 8 字节的 `cidContainer`，所以 `pContainer` 位于结构偏移：

$$
8+200=208=0xd0
$$

构造超过 100 个宽字符的名称，在偏移 `0xd0` 放入伪对象地址，即可控制 `pContainer`。伪对象首个机器字应指向伪 vftable，表项则对应 `removeContainer`、`releaseContainer` 以及后续析构路径所调用的位置。

### 从伪对象进入内核控制流

关闭日志文件时，驱动遍历所有 container：

```c
NTSTATUS closeLogFile(HANDLE file)
{
    myLOG_HEADER *logFileMem = (myLOG_HEADER *)file;

    for (int i = 0; i < logFileMem->Containers; i++) {
        my_CLFS_CONTAINER_CONTEXT *thisContainer =
            (my_CLFS_CONTAINER_CONTEXT *)(
                (unsigned long long)logFileMem
                + logFileMem->rgContainers[i]
            );

        thisContainer->pContainer->removeContainer();
        thisContainer->pContainer->releaseContainer();
        delete thisContainer->pContainer;
        memset(&thisContainer->pContainer, 0, 8);
    }

    /* 其余清理逻辑 */
}
```

因此触发顺序为：

1. 用 `addLogContainer` 的超长名称把 `pContainer` 改成伪对象；
2. 在可预测、驱动可访问的位置布置伪对象和伪 vftable；
3. 调用关闭日志文件的 IOCTL；
4. `removeContainer()` 首次虚调用从伪 vftable 取函数地址，控制 RIP；
5. 用题目驱动或内核模块中的现有 gadget 建立后续提权原语；
6. 提权完成后必须安全处理后续 `releaseContainer()` 和 `delete`，或让第一条控制流不再返回到原清理循环。

在启用 SMEP 的系统上，vftable 不能直接指向用户态 shellcode；应使用内核映像中的 ROP/JOP gadget，或先构造能修改控制寄存器、令牌或线程状态的内核原语。

### 参考文章中需要带入本题的后利用方法

[Zscaler 对 CVE-2022-37969 的 Part 2 利用分析](https://www.zscaler.com/blogs/security-research/technical-analysis-windows-clfs-zero-day-vulnerability-cve-2022-37969-part2-exploit-analysis)描述的是 Windows 原生 CLFS 漏洞，不是本题 `containerName` 溢出的成因；它在本题中的价值是提供内核堆布局和提权阶段的参考。文章的关键内容包括：

- 原 exploit 在 BigPool 中喷射 CLFS Base Block，并利用池布局把易受攻击对象放到目标对象附近；
- 后利用可以定位当前进程与 PID 4 的 `EPROCESS`，把当前进程的 Token 替换为 System Token；
- `EPROCESS.Token` 是 `EX_FAST_REF`，低 4 bit 保存引用计数，覆盖指针时需要保留或正确处理这些标志位；
- 另一条路线是把当前线程 `ETHREAD.PreviousMode` 改为 `KernelMode`，使相关系统调用不再按用户地址执行常规探测，再借此完成内核地址读写；
- 文中的结构偏移，例如测试系统上的 `EPROCESS.Token + 0x4b8` 和 `ETHREAD.PreviousMode + 0x232`，只适用于对应 Windows 构建，不能当作通用常量。

本题已经通过伪 vftable 直接给出控制流劫持入口，不需要原样重现文章中的 CLFS 池破坏；应借鉴的是“从内核执行原语到稳定令牌替换”的后半段，并根据题目虚拟机版本重新获取结构偏移和 gadget。

## 方法总结

本题的核心链路为“宽字符串越界→覆盖 C++ 对象指针→伪造对象与 vftable→内核虚调用劫持→令牌替换或等价提权”。`pContainer` 位于 `containerName` 之后，使覆盖偏移可以直接从结构体布局算出；真正需要谨慎处理的是 SMEP、Windows 版本相关偏移以及关闭路径中的连续三次对象操作。

外部 CVE 文章保留为利用技术来源，但正文已区分真实 CVE 的漏洞成因与题目漏洞，并概括了复现所需的池布局、`EPROCESS.Token`、`EX_FAST_REF` 和 `PreviousMode` 信息。原材料没有给出设备 GUID、IOCTL、内核版本和最终 ROP 链，因此这里不伪造“可直接运行”的完整提权代码。
