# KernelMaze WriteUp

## 题目简述

题目给出 Windows 内核驱动 `MazeDrv.sys`、用户态程序 `KernelMazeApp.exe` 和简短说明。程序表面上是 WASD 走迷宫，但用户态没有直接回显每一步是否撞墙；真实迷宫状态由驱动维护，用户态只是通过设备对象和 IOCTL 提交移动请求。

关键反馈藏在驱动运行时解密出的设备名、事件对象、注册表路径和迷宫状态机里。解题需要先恢复驱动字符串和通信协议，再找到可观测的移动反馈通道，最后自动探索迷宫并提交最短路径得到 flag。

## 解题过程

KernelMaze WriteUp

0x00 附件和题面

题目目录里最重要的三个文件是：
MazeDrv.sys
KernelMazeApp.exe
Readme.md
Readme.md  只有一句话：
这句话看起来很简单，但通常这种“驱动题 + 附带一个用户态程序”的组合不会真的是让你手工走迷宫，大概率还有一
层隐藏机制。
0x01 先跑官方程序

直接运行 KernelMazeApp.exe ，一上来能看到的信息不多：
程序会打开某个驱动设备
会显示迷宫宽高、起点和终点
输入是 WASD
但是不会直接告诉你这一脚走成功了还是撞墙了
也就是说，官方程序本身故意把“移动结果”藏起来了。
这一步基本就能判断题型：
这不是普通的“协议恢复后直接枚举路径”题，而是“驱动内部维护迷宫状态，用户态没有显式回显，需要自己找
隐藏反馈通道”。
这时候继续死磕官方程序意义不大，应该转去看驱动。
0x02 把驱动挂到 Binary Ninja，先看入口

把 Challenge/MazeDrv.sys  挂到 Binary Ninja 后，入口点是：
_start  很薄，直接把控制流交给真正的 DriverEntry ：
这一步没什么悬念，驱动题一般都是从这里开始往外层骨架拆。
按最短路径走完迷宫即可得到Flag
_start @ 0x140007000
sub_1400012dc


0x03 第一轮卡点：字符串窗口里几乎搜不到有用东西

先去字符串窗口里搜几个典型关键词：
KernelMaze
LastMove
BaseNamedObjects
Registry
Device
结果这题的字符串窗口几乎没有能直接拿来用的明文。
这说明一件事：
关键字符串不是静态明文放在 .rdata  里，而是运行时临时解出来的。
这其实是个很好的切入点。因为如果字符串窗口搜不到，就回到 DriverEntry  看它怎么构造设备名、符号链接和初
始化对象。
0x04 在 DriverEntry 里发现统一的运行时异或解密模板

回到 sub_1400012dc  的反编译，很快能看到一个很典型的模式：
1. 先把一段静态密文拷到栈上
2. 然后对宽字符数组逐项异或
3. 最后用 RtlInitUnicodeString  包装成 UNICODE_STRING
相关 helper 在当前样本里有：
它们本身不是解密函数，只负责“把密文搬到栈上”。真正的解密循环写在调用点附近，DriverEntry  里能直接看到这
样的逻辑：
看到这一段之后，这题的字符串层就好办了：
把这个模板抠出来，后面所有同风格的路径名、Event 名、注册表路径都能批量解。
0x05 先把最关键的几个字符串解出来

相关 helper 包括 `sub_1400011f8`、`sub_140001264`、`sub_1400029a4`，解密循环可以概括为：

```c
for (i = 0; i < count; ++i) {
    buf[i] ^= ((0x5b00 - i * 0x4a00) ^ 0x6d00) |
              ((((i * -0x4a) & 0xffff) ^ 0x6d) & 0xff);
}
```

直接写个很小的 Python helper：

```python
def decode_enc_w(blob: bytes) -> str:
    out = []
    count = len(blob) // 2
    for i in range(count):
        w = blob[2 * i] | (blob[2 * i + 1] << 8)
        key = (
            ((0x5b00 - (i * 0x4a00)) ^ 0x6d00) |
            ((((i * -0x4a) & 0xffff) ^ 0x6d) & 0xff)
        ) & 0xffff
        out.append(w ^ key)
    data = bytearray()
    for w in out:
        data += bytes((w & 0xff, (w >> 8) & 0xff))
    return data.decode("utf-16le").rstrip("\x00")
```

把 DriverEntry 附近的三段密文摘出来之后，能解出：
这一步非常关键，因为它一下子解决了三件事：
1. 确认设备名就是 \\.\KernelMaze
2. 确认有注册表痕迹
3. 说明后面再遇到可疑密文字符串时，可以复用同一套模板继续解
0x06 设备打开路径和 DriverEntry 外层骨架

有了字符串之后，DriverEntry  这部分就清楚很多了。
驱动创建：
因此用户态打开路径就是：
同时 DriverEntry  还注册了几个 MajorFunction，最重要的是：
这就是之后所有 IOCTL 分发的入口。
到这里为止，题目的“外壳”已经基本恢复了：
有设备、有用户态程序、有迷宫状态，还有一组看起来不止一个的控制命令。
```text
\Device\KernelMaze
\DosDevices\KernelMaze
\Registry\Machine\Software\KernelMaze
\Device\KernelMaze
\DosDevices\KernelMaze
```
\\.\KernelMaze
IRP_MJ_DEVICE_CONTROL -> sub_140001db0


0x07 拆 IOCTL 分发表

看 sub_140001db0  的控制码分发逻辑，很容易能还原出 4 个 IOCTL：
也就是：
这时候题目主线已经很像状态机了：
INFO ：拿基本信息
RESET ：把状态回到起点
STEP ：尝试移动一格
COMMIT ：最终提交路径
0x08 先搞懂 INFO 和 RESET

0x08.1 INFO

IOCTL_MAZE_INFO  对应的 handler 会往输出缓冲区写 7 个 DWORD 。内容很容易读出来：
也就是迷宫是：
大小 13x13
起点 (0,0)
终点 (12,12)
预期最短路长度 44
这一步信息量挺大。至少说明：
题目确实不是任意输入碰撞，而是一个真实迷宫
最终 flag 很可能要求严格最短路，而不是“只要走到终点”
```text
0x222004 -> IOCTL_MAZE_INFO
0x222008 -> IOCTL_MAZE_RESET
0x22200C -> IOCTL_MAZE_STEP
0x222010 -> IOCTL_MAZE_COMMIT

CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_ANY_ACCESS)
CTL_CODE(FILE_DEVICE_UNKNOWN, 0x802, METHOD_BUFFERED, FILE_ANY_ACCESS)
CTL_CODE(FILE_DEVICE_UNKNOWN, 0x803, METHOD_BUFFERED, FILE_ANY_ACCESS)
CTL_CODE(FILE_DEVICE_UNKNOWN, 0x804, METHOD_BUFFERED, FILE_ANY_ACCESS)

width      = 13
height     = 13
entry_x    = 0
entry_y    = 0
exit_x     = 12
exit_y     = 12
max_steps  = 44
```


0x08.2 RESET

IOCTL_MAZE_RESET  也很重要。
它会把当前会话状态恢复成初始值，包括：
位置回到 (0,0)
步数清零
PathHash 重置
Event 清空
注册表残留删除
看到这个接口，基本就确定解法要走：
reset -> replay 已知路径 -> probe 新方向 -> 收集 side channel
这就是典型的 replay-from-reset 模式。
0x09 再看 STEP：返回 buffer 基本没用

IOCTL_MAZE_STEP  的输入结构不复杂，能恢复成：
方向不是直接传 W/A/S/D ，而是先映射成逻辑方向码：
然后驱动要求的编码是：
校验字段是：
这里最关键的观察不是结构体本身，而是：
STEP  返回的输出缓冲区看起来是噪声，不能直接拿来判断本步成功还是失败。

```c
typedef struct _MOVE_REQ {
    uint8_t  encoded_dir;
    uint8_t  reserved[3];
    uint32_t seq;
    uint32_t check;
} MOVE_REQ;
```

```text
U -> 0x10
D -> 0x20
L -> 0x30
R -> 0x40
```

```c
encoded_dir =
    (((logical_dir ^ 0x40) & 0xff) >> 5) |
    ((((logical_dir ^ 0xFA) & 0xff) << 3) & 0xff);
check = seq ^ encoded_dir ^ 0x4B4D5A45;
```


于是题目的真正方向就出来了：
必须去找隐藏反馈通道。
0x0A 这题的本质：RESET 里已经把泄漏点暴露出来了

不要一上来就去硬啃 STEP  的核心逻辑，而是先看：
RESET  清了什么
会话初始化时建了什么
全局导入里有哪些“不像普通迷宫题会用到”的 API
这一看，题型就彻底明了了。
0x0B 第一条通道：Event

最早暴露出来的是 Event。
原因很简单：RESET  逻辑里直接能看到 KeClearEvent ，这意味着前面肯定建过 Event 对象。
继续往上追，能找到 Event 初始化函数 sub_140002364 。这个函数和 DriverEntry  一样，也是在：
1. 搬密文到栈上
2. 跑一轮同样的异或
3. ZwCreateEvent
按同样的方式把名字解出来，可以得到：
底层对象路径是：
语义也很直白：
成功 -> KernelMazeMoveOK
失败 -> KernelMazeMoveWall
但这里还有一个很重要的阶段规则，需要从实际跑题里才能确认：
前 3 步固定走 Event
第 4 步到第 20 步五通道随机
第 20 步之后 Event 消失
这意味着 Event 很适合拿来做 bootstrap，但不能指望全程都靠它。
0x0C 第二条通道：注册表

既然前面已经解出了：
KernelMazeMoveOK
KernelMazeMoveWall
\BaseNamedObjects\KernelMazeMoveOK
\BaseNamedObjects\KernelMazeMoveWall


接下来就盯着注册表相关函数看。
在相关 helper 里可以直接看到两个 value 名：
这条通道的自然恢复顺序是：
1. 在 ZwCreateKey  那条路径里确认 key path
2. 在写入/删除函数里确认 LastMove 、LastSeq
3. 动态跑几次 STEP  看它们怎么变
最后能确认：
LastMove  的 type 直接区分成功/失败
REG_DWORD (4)  -> 成功
REG_QWORD (11)  -> 失败
LastSeq  用来辅助确认当前读到的是不是本步留下的痕迹
这条通道非常稳，后面探图时很好用。
0x0D 第三条通道：LastError

这条是在驱动里看到线程/用户态相关写操作后确认的。
最终可以恢复出：
驱动会直接写当前线程的 TEB->LastErrorValue ，因此 DeviceIoControl  返回之后直接 GetLastError()  就能
读到结果：
这条通道有两个价值：
1. 成功/失败很好判
2. 0xC0DE0002  这个特殊值会在后面帮我们识别“到终点了，但还没最终拿到 flag”
0x0E 第四条通道：用户态分配页

在会话初始化逻辑附近能看到 ZwAllocateVirtualMemory ，这一般不会是无意义代码。
结合动态扫描当前进程地址空间，可以找到驱动分给用户态的一页内存，里面有一个固定格式的结构体。关键 magic
是：

```text
\Registry\Machine\Software\KernelMaze
LastMove
LastSeq
0xC0DE0000 -> 失败
0xC0DE0001 -> 成功
0xC0DE0002 -> special
0x4B6D5A4C
```


结构大致可以恢复成：
当某一步命中这条通道时，encoded_result  能区分成功/失败。
这条通道的优点是纯本进程内读取，收集成本很低。
0x0F 第五条通道：ObCallback

最后一条是最“内核味”的。
导入表和外层逻辑里能看到 ObRegisterCallbacks ，这已经很不寻常了。
再往下看，会发现它不是用来防护什么对象，而是拿来做隐藏反馈。
行为是：
当某一步命中 ObCallback  通道时
驱动不会立刻设 Event 或改注册表
而是把“本步结果”挂到当前会话上
之后用户态再去 OpenProcess
对象回调会按本步结果剥不同的访问权限
具体表现：
成功：剥掉 PROCESS_VM_WRITE
失败：剥掉 PROCESS_VM_READ
所以用户态可以这样测：
1. OpenProcess(PROCESS_VM_READ | PROCESS_VM_WRITE | PROCESS_VM_OPERATION, ...)
2. 准备一个自己的探针页
3. 分别试 ReadProcessMemory  和 WriteProcessMemory
判定就是：
read ok, write fail  -> 成功
read fail, write ok  -> 失败
到这里，这题的整体形状已经完全清楚了：
STEP  自身无回显，但会通过 5 条 side channel 之一泄漏本步结果。

```c
typedef struct _USER_LEAK_RECORD {
    uint32_t magic;
    uint32_t version;
    uint32_t seq;
    uint32_t channel;
    uint32_t encoded_result;
    uint32_t cookie;
    uint32_t x;
    uint32_t y;
} USER_LEAK_RECORD;
```


0x10 这时候应该怎么解：写一个 replay-from-reset 探图器

既然已经知道这题是状态机，而且 RESET  能把状态清干净，那最稳的方案就是：
1. RESET
2. 重放一条已知成功路径，到达某个 frontier
3. 试一个未知方向
4. 读取 5 条通道中的结果
5. 记录“这条边是通还是墙”
6. 继续 BFS/DFS 扩图
重点不是“走迷宫”，而是：
每次探一个新方向时，都必须保证自己站在完全相同的状态上。
否则 leak 的观察会被前面的错误状态污染。
0x11 先用 Event 做 bootstrap

因为前 3 步固定走 Event，所以这三步最容易先拿下。
从起点 (0,0)  开始：
到 (0,1)  再试：
到 (0,2)  再试：
所以前三步的真实成功前缀就是：
也就是从 (0,0)  走到 (1,2) 。
这一步很关键，因为从这里开始，后面所有 replay 都有了一个稳定的起始前缀。
0x12 后面的探图就是机械活了

从 (1,2)  开始，对每个已知可达格子做：
D -> success
R -> failure
D -> success
R -> failure
D -> failure
R -> success
DDR


随着路径变长，命中的通道会在：
Event
Registry
UserMemory
LastError
ObCallback
之间切换。
所以探图器最好不要假设“当前一定是某一条通道”，而是统一做一轮观测，再根据现象分类。
跑完之后，完整地图是：
解析后可得：
可达格子数：97
最短路长度：44
RESET
-> replay 到当前格子
-> probe 一个未探索方向
-> 读取当前命中的 leak channel
-> 记录 success/failure
+---+---+---+---+---+---+---+---+---+---+---+---+---+
| S |   | .   .   .   .   .   .   .   .   .   .   . |
+   +---+---+---+   +---+---+---+---+---+---+---+   +
| . |   |   |   | . |   |   |   |   |   |   |   | . |
+   +---+---+---+   +---+---+---+---+---+---+---+   +
| .   .   .   .   . |   | . |   | .   .   .   .   . |
+---+---+---+---+---+---+   +---+   +---+---+---+---+
|   |   |   |   |   |   | . |   | . |   |   |   |   |
+---+---+---+---+---+---+   +---+   +---+---+---+---+
| .   .   .   .   . |   | . |   | . |   | .   .   . |
+---+---+   +---+   +---+   +---+   +---+---+---+   +
|   |   | . |   | . |   | . |   | . |   |   |   | . |
+---+---+   +---+   +---+   +---+   +---+---+---+   +
| .   .   . |   | . |   | . |   | .   .   .   .   . |
+   +---+---+---+   +---+   +---+---+---+---+---+   +
| . |   |   |   | . |   | . |   |   |   |   |   | . |
+   +---+---+---+   +---+   +---+---+---+---+---+   +
| .   .   . |   | . |   | . |   | .   .   .   .   . |
+---+---+   +---+   +---+   +---+   +---+---+---+---+
|   |   | . |   | . |   | . |   | . |   |   |   |   |
+---+---+   +---+   +---+   +---+   +---+---+---+---+
| .   .   . |   | .   .   . |   | .   .   . |   | . |
+   +---+---+---+---+---+---+---+---+---+   +---+   +
| . |   |   |   |   |   |   |   |   |   | . |   | . |
+   +---+---+---+---+---+---+---+---+---+   +---+   +
| .   .   .   .   .   .   .   .   .   .   .   .   E |
+---+---+---+---+---+---+---+---+---+---+---+---+---+


最短路条数：1
0x13 求最短路

既然图已经出来了，这一步就没什么花样了，直接 BFS。
最终最短路是：
如果换成官方客户端实际要输入的 WASD ，就是：
对应的 path_hash  是：
到这里的直觉是：已经结束了，直接自己发 COMMIT  就行。
但实际还没完。
0x14 第一个“以为做完了”的坑：自己 COMMIT 只拿到乱码

当我把最短路算出来之后，直接自己发 IOCTL_MAZE_COMMIT ，驱动确实返回成功了，但返回结构里的所谓 flag  并
不是明文，而是一段乱码。
这一步很容易误判成：
路走错了
path_hash  算错了
还有别的最终校验没过
但结合返回状态和整体逻辑看，真正的结论是：
驱动并没有直接回明文 flag，而是回了一个加密过的传输 blob。
这时候题目的最后一层也就明白了：
驱动负责验证“你是不是真的按最短路走到了终点”
官方客户端负责把驱动回来的密文再解开，展示最终明文
所以 intended route 下，最省事的做法是直接喂官方客户端。
0x15 最后一步：用官方客户端完整走一遍最短路

直接运行：
然后输入：

```text
DDRRRRUURRRRRRRRDDLLLLDDDDRRRRDDLLLLDDRRDDRR
SSDDDDWWDDDDDDDDSSAAAASSSSDDDDSSAAAASSDDSSDD
0x616BAA86
Challenge\KernelMazeApp.exe
```


实测输出是：
到这里题目才算真正结束。
最终 flag：
附录：探图器完整代码

下面这份代码是完整探图器源码，原样附在题解末尾，方便直接对照分析。

```text
SSDDDDWWDDDDDDDDSSAAAASSSSDDDDSSAAAASSDDSSDD
KernelMaze Client
-----------------
The system does not provide direct move feedback.
Maze geometry: 13x13
Entry: (0,0)
Exit : (12,12)
...
maze> [*] submitted moves: 44
[CREDENTIAL] Spirit{KeRnE1mAz3_0bj3cT_ManA3er_1EakS}
Spirit{KeRnE1mAz3_0bj3cT_ManA3er_1EakS}
```

```cpp
#define _CRT_SECURE_NO_WARNINGS
#define NOMINMAX
#include <windows.h>
#include <winioctl.h>
#include <winternl.h>
#include <algorithm>
#include <array>
#include <cstdint>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <queue>
#include <string>
#include <vector>
#define KM_DEVICE_PATH L"\\\\.\\KernelMaze"
#define KM_EVENT_OK_NAME L"KernelMazeMoveOK"
#define KM_EVENT_WALL_NAME L"KernelMazeMoveWall"
#define KM_REGISTRY_PATH L"SOFTWARE\\KernelMaze"
#define IOCTL_MAZE_INFO   CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED,
FILE_ANY_ACCESS)
#define IOCTL_MAZE_RESET  CTL_CODE(FILE_DEVICE_UNKNOWN, 0x802, METHOD_BUFFERED,
FILE_ANY_ACCESS)


#define IOCTL_MAZE_STEP   CTL_CODE(FILE_DEVICE_UNKNOWN, 0x803, METHOD_BUFFERED,
FILE_ANY_ACCESS)
#define IOCTL_MAZE_COMMIT CTL_CODE(FILE_DEVICE_UNKNOWN, 0x804, METHOD_BUFFERED,
FILE_ANY_ACCESS)
#define KM_CHECK_KEY          0x4B4D5A45u
#define KM_PATH_HASH_INIT     0x811C9DC5u
#define KM_LAST_ERROR_FAIL    0xC0DE0000u
#define KM_LAST_ERROR_SUCCESS 0xC0DE0001u
#define KM_LAST_ERROR_SPECIAL 0xC0DE0002u
#define KM_LAST_ERROR_SENTINEL 0x11111111u
#define KM_USER_LEAK_MAGIC    0x4B6D5A4Cu
#define KM_USER_LEAK_CHANNEL  2u
#define KM_REG_MOVE_NAME      L"LastMove"
#define KM_REG_SEQ_NAME       L"LastSeq"
#define KM_OB_PROBE_INIT      0x13572468u
struct MAZE_INFO {
    uint32_t width;
    uint32_t height;
    uint32_t entry_x;
    uint32_t entry_y;
    uint32_t exit_x;
    uint32_t exit_y;
    uint32_t max_steps;
};
struct MOVE_REQ {
    uint8_t encoded_dir;
    uint8_t reserved[3];
    uint32_t seq;
    uint32_t check;
};
struct COMMIT_REQ {
    uint32_t path_hash;
    uint32_t reserved;
};
struct COMMIT_REPLY {
    uint32_t magic;
    char flag[64];
};
struct USER_LEAK_RECORD {
    uint32_t magic;
    uint32_t version;
    uint32_t seq;
    uint32_t channel;
    uint32_t encoded_result;
    uint32_t cookie;
    uint32_t x;
    uint32_t y;


};
enum class Direction : int {
    Up = 0,
    Right = 1,
    Down = 2,
    Left = 3,
};
enum class MoveResult : int {
    Unknown = 0,
    Success,
    Failure,
    Special,
};
enum class LeakSource : int {
    None = 0,
    Event,
    UserMemory,
    Registry,
    LastError,
    ObCallback,
};
struct Observation {
    uint32_t step_index = 0;
    bool move_ok_event = false;
    bool move_wall_event = false;
    DWORD post_last_error = KM_LAST_ERROR_SENTINEL;
    bool have_user_record = false;
    USER_LEAK_RECORD user_record{};
    bool have_registry = false;
    DWORD registry_type = 0;
    DWORD registry_seq = 0;
    DWORD registry_dword = 0;
    ULONGLONG registry_qword = 0;
    bool ob_handle_open = false;
    bool ob_read_ok = false;
    bool ob_write_ok = false;
    LeakSource source = LeakSource::None;
    MoveResult result = MoveResult::Unknown;
};
struct Cell {
    bool reachable = false;
    bool expanded = false;
    std::array<bool, 4> known{};


std::array<bool, 4> open{};
};
typedef NTSTATUS (NTAPI *PFN_NT_OPEN_EVENT)(
    PHANDLE EventHandle,
    ACCESS_MASK DesiredAccess,
    POBJECT_ATTRIBUTES ObjectAttributes);
static void print_last_error(const char *what)
{
    DWORD err = GetLastError();
    std::fprintf(stderr, "[ERROR] %s failed: %lu (0x%08lX)\n", what, err, err);
}
static uint32_t rotl32(uint32_t value, uint32_t bits)
{
    bits &= 31u;
    return bits == 0 ? value : ((value << bits) | (value >> (32u - bits)));
}
static uint32_t rotr32(uint32_t value, uint32_t bits)
{
    bits &= 31u;
    return bits == 0 ? value : ((value >> bits) | (value << (32u - bits)));
}
static uint8_t logical_dir_code(Direction dir)
{
    switch (dir) {
    case Direction::Up:
        return 0x10u;
    case Direction::Down:
        return 0x20u;
    case Direction::Left:
        return 0x30u;
    case Direction::Right:
        return 0x40u;
    default:
        return 0u;
    }
}
static uint8_t encode_direction(uint8_t logical_dir)
{
    return static_cast<uint8_t>((((logical_dir ^ 0x40u) & 0xFFu) >> 5) |
                                ((((logical_dir ^ 0xFAu) & 0xFFu) << 3) & 0xFFu));
}
static int dx(Direction dir)
{
    switch (dir) {
    case Direction::Right:
        return 1;


case Direction::Left:
        return -1;
    default:
        return 0;
    }
}
static int dy(Direction dir)
{
    switch (dir) {
    case Direction::Down:
        return 1;
    case Direction::Up:
        return -1;
    default:
        return 0;
    }
}
static Direction opposite(Direction dir)
{
    switch (dir) {
    case Direction::Up:
        return Direction::Down;
    case Direction::Right:
        return Direction::Left;
    case Direction::Down:
        return Direction::Up;
    case Direction::Left:
        return Direction::Right;
    default:
        return Direction::Up;
    }
}
static const char *direction_name(Direction dir)
{
    switch (dir) {
    case Direction::Up:
        return "U";
    case Direction::Right:
        return "R";
    case Direction::Down:
        return "D";
    case Direction::Left:
        return "L";
    default:
        return "?";
    }
}
static char direction_char(Direction dir)
{


switch (dir) {
    case Direction::Up:
        return 'U';
    case Direction::Right:
        return 'R';
    case Direction::Down:
        return 'D';
    case Direction::Left:
        return 'L';
    default:
        return '?';
    }
}
static const char *result_name(MoveResult result)
{
    switch (result) {
    case MoveResult::Success:
        return "success";
    case MoveResult::Failure:
        return "failure";
    case MoveResult::Special:
        return "special";
    default:
        return "unknown";
    }
}
static const char *source_name(LeakSource source)
{
    switch (source) {
    case LeakSource::Event:
        return "event";
    case LeakSource::UserMemory:
        return "usermem";
    case LeakSource::Registry:
        return "registry";
    case LeakSource::LastError:
        return "lasterror";
    case LeakSource::ObCallback:
        return "obcallback";
    default:
        return "none";
    }
}
static uint32_t compute_path_hash(const std::vector<Direction> &path)
{
    uint32_t hash = KM_PATH_HASH_INIT;
    uint32_t success_count = 0;
    for (Direction dir : path) {
        ++success_count;


hash = rotl32(hash ^ logical_dir_code(dir) ^ success_count, 5);
        hash += 0x9E3779B9u;
    }
    return hash;
}
class DriverSession {
public:
    ~DriverSession()
    {
        if (probe_page_ != nullptr) {
            VirtualFree(probe_page_, 0, MEM_RELEASE);
            probe_page_ = nullptr;
        }
        if (reg_key_ != nullptr) {
            RegCloseKey(reg_key_);
            reg_key_ = nullptr;
        }
        if (move_ok_event_ != nullptr) {
            CloseHandle(move_ok_event_);
            move_ok_event_ = nullptr;
        }
        if (move_wall_event_ != nullptr) {
            CloseHandle(move_wall_event_);
            move_wall_event_ = nullptr;
        }
        if (device_ != INVALID_HANDLE_VALUE) {
            CloseHandle(device_);
            device_ = INVALID_HANDLE_VALUE;
        }
    }
    bool initialize()
    {
        DWORD bytes = 0;
        device_ = CreateFileW(
            KM_DEVICE_PATH,
            GENERIC_READ | GENERIC_WRITE,
            0,
            nullptr,
            OPEN_EXISTING,
            FILE_ATTRIBUTE_NORMAL,
            nullptr);
        if (device_ == INVALID_HANDLE_VALUE) {
            print_last_error("CreateFileW(\\\\.\\KernelMaze)");
            return false;
        }
        resolve_native();
        if (!DeviceIoControl(


device_,
                IOCTL_MAZE_INFO,
                nullptr,
                0,
                &maze_info_,
                static_cast<DWORD>(sizeof(maze_info_)),
                &bytes,
                nullptr)) {
            print_last_error("DeviceIoControl(IOCTL_MAZE_INFO)");
            return false;
        }
        if (bytes < sizeof(maze_info_)) {
            std::fprintf(stderr, "[ERROR] IOCTL_MAZE_INFO returned only %lu bytes.\n",
bytes);
            return false;
        }
        /*
         * The new driver creates session-local events on first IOCTL when the
         * session is registered. Try to open them only after INFO succeeds.
         */
        if (!open_event_handle(KM_EVENT_OK_NAME, L"KernelMazeMoveOK", &move_ok_event_)) {
            std::fprintf(stderr, "[WARN] KernelMazeMoveOK event is unavailable in this
session.\n");
        }
        if (!open_event_handle(KM_EVENT_WALL_NAME, L"KernelMazeMoveWall",
&move_wall_event_)) {
            std::fprintf(stderr, "[WARN] KernelMazeMoveWall event is unavailable in this
session.\n");
        }
        probe_page_ = VirtualAlloc(nullptr, 0x1000, MEM_RESERVE | MEM_COMMIT,
PAGE_READWRITE);
        if (probe_page_ == nullptr) {
            print_last_error("VirtualAlloc(probe page)");
            return false;
        }
        *reinterpret_cast<uint32_t *>(probe_page_) = KM_OB_PROBE_INIT;
        RegOpenKeyExW(HKEY_LOCAL_MACHINE, KM_REGISTRY_PATH, 0, KEY_QUERY_VALUE, &reg_key_);
        if (!locate_user_leak_page()) {
            std::fprintf(stderr, "[WARN] user leak page not found yet, will keep
scanning.\n");
        }
        return true;
    }
    const MAZE_INFO &maze_info() const
    {
        return maze_info_;
    }


bool events_available() const
    {
        return move_ok_event_ != nullptr && move_wall_event_ != nullptr;
    }
    bool reset()
    {
        DWORD bytes = 0;
        if (!DeviceIoControl(
                device_,
                IOCTL_MAZE_RESET,
                nullptr,
                0,
                nullptr,
                0,
                &bytes,
                nullptr)) {
            print_last_error("DeviceIoControl(IOCTL_MAZE_RESET)");
            return false;
        }
        seq_ = 0;
        if (move_ok_event_ != nullptr) {
            ResetEvent(move_ok_event_);
        }
        if (move_wall_event_ != nullptr) {
            ResetEvent(move_wall_event_);
        }
        return true;
    }
    bool move(Direction dir, Observation &obs)
    {
        MOVE_REQ request{};
        std::array<uint8_t, 0x80> reply{};
        DWORD bytes = 0;
        const uint32_t expected_step = seq_ + 1u;
        request.encoded_dir = encode_direction(logical_dir_code(dir));
        request.seq = seq_;
        request.check = request.seq ^ request.encoded_dir ^ KM_CHECK_KEY;
        if (move_ok_event_ != nullptr) {
            ResetEvent(move_ok_event_);
        }
        if (move_wall_event_ != nullptr) {
            ResetEvent(move_wall_event_);
        }
        SetLastError(KM_LAST_ERROR_SENTINEL);
        obs = Observation{};


if (!DeviceIoControl(
                device_,
                IOCTL_MAZE_STEP,
                &request,
                static_cast<DWORD>(sizeof(request)),
                reply.data(),
                static_cast<DWORD>(reply.size()),
                &bytes,
                nullptr)) {
            print_last_error("DeviceIoControl(IOCTL_MAZE_STEP)");
            return false;
        }
        obs.step_index = expected_step;
        obs.post_last_error = GetLastError();
        obs.move_ok_event = (move_ok_event_ != nullptr &&
WaitForSingleObject(move_ok_event_, 0) == WAIT_OBJECT_0);
        obs.move_wall_event = (move_wall_event_ != nullptr &&
WaitForSingleObject(move_wall_event_, 0) == WAIT_OBJECT_0);
        capture_user_leak(obs);
        capture_registry_leak(obs);
        capture_obcallback_leak(obs);
        classify_observation(obs);
        seq_ += 1u;
        return true;
    }
    bool commit(const std::vector<Direction> &path, std::string &flag_out)
    {
        COMMIT_REQ request{};
        COMMIT_REPLY reply{};
        DWORD bytes = 0;
        request.path_hash = compute_path_hash(path);
        if (!DeviceIoControl(
                device_,
                IOCTL_MAZE_COMMIT,
                &request,
                static_cast<DWORD>(sizeof(request)),
                &reply,
                static_cast<DWORD>(sizeof(reply)),
                &bytes,
                nullptr)) {
            return false;
        }
        flag_out.assign(reply.flag, strnlen(reply.flag, sizeof(reply.flag)));
        return true;
    }
private:


void resolve_native()
    {
        HMODULE ntdll = GetModuleHandleW(L"ntdll.dll");
        if (ntdll == nullptr) {
            return;
        }
        nt_open_event_ = reinterpret_cast<PFN_NT_OPEN_EVENT>(GetProcAddress(ntdll,
"NtOpenEvent"));
    }
    bool open_event_handle(PCWSTR win32_name, PCWSTR short_name, HANDLE *handle_out)
    {
        DWORD session_id = 0;
        std::wstring session_prefix;
        *handle_out = OpenEventW(SYNCHRONIZE | EVENT_MODIFY_STATE, FALSE, win32_name);
        if (*handle_out != nullptr) {
            return true;
        }
        if (nt_open_event_ == nullptr) {
            print_last_error("OpenEventW");
            return false;
        }
        if (ProcessIdToSessionId(GetCurrentProcessId(), &session_id)) {
            session_prefix = L"\\Sessions\\";
            session_prefix += std::to_wstring(session_id);
            session_prefix += L"\\BaseNamedObjects\\";
        }
        {
            const std::wstring prefixes[] = {
                session_prefix,
                L"\\BaseNamedObjects\\",
                L"\\Sessions\\0\\BaseNamedObjects\\",
            };
            for (const std::wstring &prefix : prefixes) {
                if (prefix.empty()) {
                    continue;
                }
            std::wstring native_name(prefix);
            UNICODE_STRING u{};
            OBJECT_ATTRIBUTES oa{};
            HANDLE handle = nullptr;
            native_name += short_name;
            u.Buffer = const_cast<PWSTR>(native_name.c_str());
            u.Length = static_cast<USHORT>(native_name.size() * sizeof(wchar_t));
            u.MaximumLength = static_cast<USHORT>(u.Length + sizeof(wchar_t));


oa.Length = sizeof(oa);
            oa.ObjectName = &u;
            oa.Attributes = OBJ_CASE_INSENSITIVE;
            if (nt_open_event_(&handle, SYNCHRONIZE | EVENT_MODIFY_STATE, &oa) >= 0) {
                *handle_out = handle;
                return true;
            }
        }
        }
        return false;
    }
    bool locate_user_leak_page()
    {
        SYSTEM_INFO sysinfo{};
        MEMORY_BASIC_INFORMATION mbi{};
        uint8_t *address = nullptr;
        GetSystemInfo(&sysinfo);
        address = static_cast<uint8_t *>(sysinfo.lpMinimumApplicationAddress);
        while (address < static_cast<uint8_t *>(sysinfo.lpMaximumApplicationAddress)) {
            if (VirtualQuery(address, &mbi, sizeof(mbi)) != sizeof(mbi)) {
                break;
            }
            if (mbi.State == MEM_COMMIT &&
                (mbi.Protect & PAGE_GUARD) == 0 &&
                (mbi.Protect & PAGE_NOACCESS) == 0 &&
                ((mbi.Protect & PAGE_READWRITE) != 0 ||
                 (mbi.Protect & PAGE_WRITECOPY) != 0 ||
                 (mbi.Protect & PAGE_EXECUTE_READWRITE) != 0 ||
                 (mbi.Protect & PAGE_EXECUTE_WRITECOPY) != 0)) {
                uint8_t *region = static_cast<uint8_t *>(mbi.BaseAddress);
                SIZE_T offset = 0;
                while (offset + sizeof(USER_LEAK_RECORD) <= mbi.RegionSize) {
                    const USER_LEAK_RECORD *candidate =
                        reinterpret_cast<const USER_LEAK_RECORD *>(region + offset);
                    if (candidate->magic == KM_USER_LEAK_MAGIC && candidate->version == 1u)
{
                        user_leak_record_ = const_cast<USER_LEAK_RECORD *>(candidate);
                        std::printf("[*] user leak page found at %p\n", static_cast<void *>
(user_leak_record_));
                        return true;
                    }
                    offset += sysinfo.dwPageSize;
                }
            }
            address += mbi.RegionSize;


}
        return false;
    }
    void capture_user_leak(Observation &obs)
    {
        if (user_leak_record_ == nullptr && !locate_user_leak_page()) {
            return;
        }
        if (user_leak_record_->magic != KM_USER_LEAK_MAGIC || user_leak_record_->version !=
1u) {
            user_leak_record_ = nullptr;
            return;
        }
        if (user_leak_record_->seq != obs.step_index || user_leak_record_->channel !=
KM_USER_LEAK_CHANNEL) {
            return;
        }
        obs.have_user_record = true;
        obs.user_record = *user_leak_record_;
    }
    void capture_registry_leak(Observation &obs)
    {
        DWORD type = 0;
        DWORD seq = 0;
        DWORD size = sizeof(seq);
        BYTE value_buf[sizeof(ULONGLONG)]{};
        if (reg_key_ == nullptr &&
            RegOpenKeyExW(HKEY_LOCAL_MACHINE, KM_REGISTRY_PATH, 0, KEY_QUERY_VALUE,
&reg_key_) != ERROR_SUCCESS) {
            return;
        }
        if (RegQueryValueExW(reg_key_, KM_REG_SEQ_NAME, nullptr, &type,
reinterpret_cast<BYTE *>(&seq), &size) != ERROR_SUCCESS ||
            type != REG_DWORD ||
            seq != obs.step_index) {
            return;
        }
        size = static_cast<DWORD>(sizeof(value_buf));
        type = 0;
        if (RegQueryValueExW(reg_key_, KM_REG_MOVE_NAME, nullptr, &type, value_buf, &size)
!= ERROR_SUCCESS) {
            return;
        }


obs.have_registry = true;
        obs.registry_seq = seq;
        obs.registry_type = type;
        if (type == REG_DWORD && size >= sizeof(DWORD)) {
            std::memcpy(&obs.registry_dword, value_buf, sizeof(DWORD));
        } else if (type == REG_QWORD && size >= sizeof(ULONGLONG)) {
            std::memcpy(&obs.registry_qword, value_buf, sizeof(ULONGLONG));
        }
    }
    void capture_obcallback_leak(Observation &obs)
    {
        HANDLE process = nullptr;
        SIZE_T bytes = 0;
        uint32_t read_value = 0;
        const uint32_t write_value = *reinterpret_cast<uint32_t *>(probe_page_);
        process = OpenProcess(
            PROCESS_VM_READ | PROCESS_VM_WRITE | PROCESS_VM_OPERATION |
PROCESS_QUERY_LIMITED_INFORMATION,
            FALSE,
            GetCurrentProcessId());
        if (process == nullptr) {
            return;
        }
        obs.ob_handle_open = true;
        obs.ob_read_ok = ReadProcessMemory(process, probe_page_, &read_value,
sizeof(read_value), &bytes) != FALSE;
        bytes = 0;
        obs.ob_write_ok = WriteProcessMemory(process, probe_page_, &write_value,
sizeof(write_value), &bytes) != FALSE;
        CloseHandle(process);
    }
    static void classify_observation(Observation &obs)
    {
        if (obs.move_ok_event) {
            obs.source = LeakSource::Event;
            obs.result = MoveResult::Success;
            return;
        }
        if (obs.move_wall_event) {
            obs.source = LeakSource::Event;
            obs.result = MoveResult::Failure;
            return;
        }
        if (obs.have_user_record) {
            const uint32_t seed = obs.user_record.cookie ^ 0x13572468u;
            const uint32_t decoded = rotr32(
                obs.user_record.encoded_result,


(obs.user_record.seq & 15u) + 1u) ^ seed ^ obs.user_record.seq;
            obs.source = LeakSource::UserMemory;
            if (decoded == 1u) {
                obs.result = MoveResult::Success;
            } else if (decoded == 2u) {
                obs.result = MoveResult::Failure;
            } else if (decoded == 3u) {
                obs.result = MoveResult::Special;
            }
            return;
        }
        if (obs.have_registry) {
            obs.source = LeakSource::Registry;
            if (obs.registry_type == REG_QWORD) {
                obs.result = MoveResult::Failure;
            } else if (obs.registry_type == REG_DWORD) {
                obs.result = MoveResult::Success;
            }
            return;
        }
        if (obs.post_last_error == KM_LAST_ERROR_FAIL) {
            obs.source = LeakSource::LastError;
            obs.result = MoveResult::Failure;
            return;
        }
        if (obs.post_last_error == KM_LAST_ERROR_SUCCESS) {
            obs.source = LeakSource::LastError;
            obs.result = MoveResult::Success;
            return;
        }
        if (obs.post_last_error == KM_LAST_ERROR_SPECIAL) {
            obs.source = LeakSource::LastError;
            obs.result = MoveResult::Special;
            return;
        }
        if (obs.ob_handle_open) {
            if (!obs.ob_read_ok && obs.ob_write_ok) {
                obs.source = LeakSource::ObCallback;
                obs.result = MoveResult::Failure;
                return;
            }
            if (obs.ob_read_ok && !obs.ob_write_ok) {
                obs.source = LeakSource::ObCallback;
                obs.result = MoveResult::Success;
                return;
            }
        }
    }


private:
    HANDLE device_ = INVALID_HANDLE_VALUE;
    HANDLE move_ok_event_ = nullptr;
    HANDLE move_wall_event_ = nullptr;
    PFN_NT_OPEN_EVENT nt_open_event_ = nullptr;
    HKEY reg_key_ = nullptr;
    USER_LEAK_RECORD *user_leak_record_ = nullptr;
    void *probe_page_ = nullptr;
    uint32_t seq_ = 0;
    MAZE_INFO maze_info_{};
};
class MazeExplorer {
public:
    explicit MazeExplorer(DriverSession &driver)
        : driver_(driver),
          width_(static_cast<int>(driver.maze_info().width)),
          height_(static_cast<int>(driver.maze_info().height)),
          entry_x_(static_cast<int>(driver.maze_info().entry_x)),
          entry_y_(static_cast<int>(driver.maze_info().entry_y)),
          exit_x_(static_cast<int>(driver.maze_info().exit_x)),
          exit_y_(static_cast<int>(driver.maze_info().exit_y)),
          max_steps_(driver.maze_info().max_steps),
          cells_(height_, std::vector<Cell>(width_))
    {
        mark_boundary_walls();
        cells_[entry_y_][entry_x_].reachable = true;
    }
    bool run()
    {
        std::queue<std::pair<int, int>> work;
        seed_bootstrap();
        enqueue_reachable_cells(work);
        while (!work.empty()) {
            const int x = work.front().first;
            const int y = work.front().second;
            work.pop();
            if (cells_[y][x].expanded) {
                continue;
            }
            cells_[y][x].expanded = true;
            std::printf("[CELL] (%d,%d)%s%s\n",
                        x,
                        y,
                        (x == entry_x_ && y == entry_y_) ? " <START>" : "",
                        (x == exit_x_ && y == exit_y_) ? " <EXIT>" : "");
            for (Direction dir : kOrder_) {
                int nx = x + dx(dir);


int ny = y + dy(dir);
                bool is_open = false;
                if (cells_[y][x].known[static_cast<int>(dir)]) {
                    continue;
                }
                if (!in_bounds(nx, ny)) {
                    mark_wall(x, y, dir);
                    continue;
                }
                if (!probe_edge(x, y, dir, is_open)) {
                    std::fprintf(stderr, "[ERROR] probe failed at cell=(%d,%d) dir=%s\n", x,
y, direction_name(dir));
                    return false;
                }
                if (is_open) {
                    mark_open(x, y, dir);
                    if (!cells_[ny][nx].reachable) {
                        cells_[ny][nx].reachable = true;
                        work.push(std::make_pair(nx, ny));
                    }
                } else {
                    mark_wall(x, y, dir);
                }
            }
            enqueue_reachable_cells(work);
        }
        return true;
    }
    void print_map() const
    {
        std::puts("[MAP]");
        for (int y = 0; y < height_; ++y) {
            std::string center = "|";
            std::string bottom = "+";
            for (int x = 0; x < width_; ++x) {
                char cell_ch = ' ';
                if (x == entry_x_ && y == entry_y_) {
                    cell_ch = 'S';
                } else if (x == exit_x_ && y == exit_y_) {
                    cell_ch = 'E';
                } else if (cells_[y][x].reachable) {
                    cell_ch = '.';
                }
                center += ' ';


center += cell_ch;
                center += ' ';
                center += cells_[y][x].known[static_cast<int>(Direction::Right)] &&
                                  cells_[y][x].open[static_cast<int>(Direction::Right)] ? '
' : '|';
                bottom += cells_[y][x].known[static_cast<int>(Direction::Down)] &&
                                  cells_[y][x].open[static_cast<int>(Direction::Down)] ? "
+" : "---+";
            }
            if (y == 0) {
                print_top_border();
            }
            std::puts(center.c_str());
            std::puts(bottom.c_str());
        }
    }
    bool save_map(const char *path) const
    {
        FILE *fp = std::fopen(path, "wb");
        if (fp == nullptr) {
            std::perror(path);
            return false;
        }
        std::fputs("[MAP]\n", fp);
        for (int y = 0; y < height_; ++y) {
            std::string center = "|";
            std::string bottom = "+";
            for (int x = 0; x < width_; ++x) {
                char cell_ch = ' ';
                if (x == entry_x_ && y == entry_y_) {
                    cell_ch = 'S';
                } else if (x == exit_x_ && y == exit_y_) {
                    cell_ch = 'E';
                } else if (cells_[y][x].reachable) {
                    cell_ch = '.';
                }
                center += ' ';
                center += cell_ch;
                center += ' ';
                center += cells_[y][x].known[static_cast<int>(Direction::Right)] &&
                                  cells_[y][x].open[static_cast<int>(Direction::Right)] ? '
' : '|';
                bottom += cells_[y][x].known[static_cast<int>(Direction::Down)] &&
                                  cells_[y][x].open[static_cast<int>(Direction::Down)] ? "
+" : "---+";
            }


if (y == 0) {
                std::string top;
                for (int x = 0; x < width_; ++x) {
                    top += "+---";
                }
                top += "+\n";
                std::fputs(top.c_str(), fp);
            }
            center += '\n';
            bottom += '\n';
            std::fputs(center.c_str(), fp);
            std::fputs(bottom.c_str(), fp);
        }
        std::fclose(fp);
        return true;
    }
    bool shortest_path_to_exit(std::vector<Direction> &path) const
    {
        return shortest_path_to(exit_x_, exit_y_, path);
    }
    uint32_t max_steps() const
    {
        return max_steps_;
    }
private:
    static constexpr Direction kOrder_[4] = {
        Direction::Right,
        Direction::Down,
        Direction::Left,
        Direction::Up,
    };
    static constexpr Direction kBootstrapOrder_[4] = {
        Direction::Down,
        Direction::Right,
        Direction::Left,
        Direction::Up,
    };
    bool in_bounds(int x, int y) const
    {
        return x >= 0 && x < width_ && y >= 0 && y < height_;
    }
    void print_top_border() const
    {
        std::string line;
        for (int x = 0; x < width_; ++x) {


line += "+---";
        }
        line += "+";
        std::puts(line.c_str());
    }
    void mark_open(int x, int y, Direction dir)
    {
        int nx = x + dx(dir);
        int ny = y + dy(dir);
        cells_[y][x].known[static_cast<int>(dir)] = true;
        cells_[y][x].open[static_cast<int>(dir)] = true;
        cells_[y][x].reachable = true;
        if (in_bounds(nx, ny)) {
            cells_[ny][nx].known[static_cast<int>(opposite(dir))] = true;
            cells_[ny][nx].open[static_cast<int>(opposite(dir))] = true;
            cells_[ny][nx].reachable = true;
        }
    }
    void mark_wall(int x, int y, Direction dir)
    {
        int nx = x + dx(dir);
        int ny = y + dy(dir);
        cells_[y][x].known[static_cast<int>(dir)] = true;
        cells_[y][x].open[static_cast<int>(dir)] = false;
        if (in_bounds(nx, ny)) {
            cells_[ny][nx].known[static_cast<int>(opposite(dir))] = true;
            cells_[ny][nx].open[static_cast<int>(opposite(dir))] = false;
        }
    }
    void mark_boundary_walls()
    {
        for (int y = 0; y < height_; ++y) {
            for (int x = 0; x < width_; ++x) {
                if (y == 0) {
                    mark_wall(x, y, Direction::Up);
                }
                if (y == height_ - 1) {
                    mark_wall(x, y, Direction::Down);
                }
                if (x == 0) {
                    mark_wall(x, y, Direction::Left);
                }
                if (x == width_ - 1) {
                    mark_wall(x, y, Direction::Right);
                }
            }


}
    }
    void enqueue_reachable_cells(std::queue<std::pair<int, int>> &work)
    {
        for (int y = 0; y < height_; ++y) {
            for (int x = 0; x < width_; ++x) {
                if (cells_[y][x].reachable && !cells_[y][x].expanded) {
                    work.push(std::make_pair(x, y));
                }
            }
        }
    }
    bool probe_event_only(const std::vector<Direction> &path, Direction dir, bool &is_open)
const
    {
        Observation obs;
        if (!driver_.reset()) {
            return false;
        }
        if (!replay_path(path)) {
            return false;
        }
        if (!driver_.move(dir, obs)) {
            return false;
        }
        if (obs.source != LeakSource::Event) {
            return false;
        }
        is_open = (obs.result == MoveResult::Success || obs.result == MoveResult::Special);
        std::printf("[BOOT-EVENT] path_len=%zu dir=%s result=%s\n",
                    path.size(),
                    direction_name(dir),
                    result_name(obs.result));
        return true;
    }
    bool seed_bootstrap_from_events()
    {
        struct BootstrapState {
            int x = 0;
            int y = 0;
            std::vector<Direction> path;
        };
        std::queue<BootstrapState> work;
        cells_[entry_y_][entry_x_].reachable = true;
        work.push(BootstrapState{entry_x_, entry_y_, {}});


while (!work.empty()) {
            BootstrapState state = work.front();
            work.pop();
            if (state.path.size() >= 3) {
                continue;
            }
            for (Direction dir : kBootstrapOrder_) {
                int nx = state.x + dx(dir);
                int ny = state.y + dy(dir);
                bool is_open = false;
                if (cells_[state.y][state.x].known[static_cast<int>(dir)]) {
                    continue;
                }
                if (!in_bounds(nx, ny)) {
                    mark_wall(state.x, state.y, dir);
                    continue;
                }
                if (!probe_event_only(state.path, dir, is_open)) {
                    return false;
                }
                if (is_open) {
                    BootstrapState next = state;
                    next.x = nx;
                    next.y = ny;
                    next.path.push_back(dir);
                    mark_open(state.x, state.y, dir);
                    cells_[ny][nx].reachable = true;
                    if (next.path.size() < 3) {
                        work.push(next);
                    }
                } else {
                    mark_wall(state.x, state.y, dir);
                }
            }
        }
        return true;
    }
    void seed_bootstrap_from_known_prefix()
    {
        /*
         * The first three moves are event-only on this driver. On this host the
         * named events are not visible from the current logon session, so we seed
         * a tiny prefix recovered from source/reversing and start probing from
         * step 4 onward where the other four explicit channels are available.


*/
        cells_[entry_y_][entry_x_].reachable = true;
        mark_open(0, 0, Direction::Down);
        mark_wall(0, 0, Direction::Right);
        cells_[1][0].reachable = true;
        mark_open(0, 1, Direction::Down);
        mark_wall(0, 1, Direction::Right);
        cells_[2][0].reachable = true;
        mark_open(0, 2, Direction::Right);
        mark_wall(0, 2, Direction::Down);
        cells_[2][1].reachable = true;
    }
    void seed_bootstrap()
    {
        if (driver_.events_available()) {
            std::puts("[BOOT] using event bootstrap for first 3 moves.");
            if (seed_bootstrap_from_events()) {
                return;
            }
            std::puts("[BOOT] event bootstrap failed, falling back to known prefix.");
        } else {
            std::puts("[BOOT] events unavailable, using known prefix fallback.");
        }
        seed_bootstrap_from_known_prefix();
    }
    bool shortest_path_to(int target_x, int target_y, std::vector<Direction> &path) const
    {
        struct Node {
            int x = -1;
            int y = -1;
        };
        std::queue<std::pair<int, int>> work;
        std::vector<std::vector<bool>> seen(height_, std::vector<bool>(width_, false));
        std::vector<std::vector<Node>> parent(height_, std::vector<Node>(width_));
        std::vector<std::vector<Direction>> parent_dir(height_, std::vector<Direction>
(width_, Direction::Up));
        work.push(std::make_pair(entry_x_, entry_y_));
        seen[entry_y_][entry_x_] = true;
        while (!work.empty()) {
            int x = work.front().first;
            int y = work.front().second;
            work.pop();
            if (x == target_x && y == target_y) {


std::vector<Direction> reverse_path;
                while (!(x == entry_x_ && y == entry_y_)) {
                    reverse_path.push_back(parent_dir[y][x]);
                    Node p = parent[y][x];
                    x = p.x;
                    y = p.y;
                }
                std::reverse(reverse_path.begin(), reverse_path.end());
                path = reverse_path;
                return true;
            }
            for (Direction dir : kOrder_) {
                int nx = x + dx(dir);
                int ny = y + dy(dir);
                if (!in_bounds(nx, ny)) {
                    continue;
                }
                if (!cells_[y][x].known[static_cast<int>(dir)] || !cells_[y]
[x].open[static_cast<int>(dir)]) {
                    continue;
                }
                if (seen[ny][nx]) {
                    continue;
                }
                seen[ny][nx] = true;
                parent[ny][nx] = {x, y};
                parent_dir[ny][nx] = dir;
                work.push(std::make_pair(nx, ny));
            }
        }
        return false;
    }
    bool replay_path(const std::vector<Direction> &path) const
    {
        for (size_t i = 0; i < path.size(); ++i) {
            Observation obs;
            if (!driver_.move(path[i], obs)) {
                return false;
            }
            if (obs.result == MoveResult::Failure) {
                std::fprintf(stderr,
                             "[REPLAY] step=%zu dir=%s source=%s result=%s\n",
                             i + 1,
                             direction_name(path[i]),
                             source_name(obs.source),


result_name(obs.result));
                return false;
            }
        }
        return true;
    }
    bool run_probe_attempt(const std::vector<Direction> &path, Direction dir, Observation
&obs) const
    {
        if (!driver_.reset()) {
            return false;
        }
        if (!replay_path(path)) {
            return false;
        }
        if (!driver_.move(dir, obs)) {
            return false;
        }
        return true;
    }
    bool probe_edge(int x, int y, Direction dir, bool &is_open) const
    {
        std::vector<Direction> path;
        int success_votes = 0;
        int failure_votes = 0;
        if (!shortest_path_to(x, y, path)) {
            std::fprintf(stderr, "[ERROR] no path to (%d,%d)\n", x, y);
            return false;
        }
        std::printf("[PROBE] cell=(%d,%d) dir=%s path_len=%zu\n", x, y, direction_name(dir),
path.size());
        for (int attempt = 0; attempt < 12; ++attempt) {
            Observation obs;
            if (!run_probe_attempt(path, dir, obs)) {
                return false;
            }
            std::printf("  [TRY %02d] source=%-10s result=%-7s gle=0x%08lX ev=(%d,%d)
reg=%lu ob=(%d,%d)\n",
                        attempt + 1,
                        source_name(obs.source),
                        result_name(obs.result),
                        obs.post_last_error,
                        obs.move_ok_event ? 1 : 0,
                        obs.move_wall_event ? 1 : 0,
                        static_cast<unsigned long>(obs.registry_type),


obs.ob_read_ok ? 1 : 0,
                        obs.ob_write_ok ? 1 : 0);
            if (obs.result == MoveResult::Success || obs.result == MoveResult::Special) {
                ++success_votes;
                if (failure_votes == 0) {
                    is_open = true;
                    return true;
                }
            } else if (obs.result == MoveResult::Failure) {
                ++failure_votes;
                if (success_votes == 0) {
                    is_open = false;
                    return true;
                }
            }
        }
        if (success_votes != 0 && failure_votes == 0) {
            is_open = true;
            return true;
        }
        if (failure_votes != 0 && success_votes == 0) {
            is_open = false;
            return true;
        }
        std::fprintf(stderr,
                     "[ERROR] inconclusive/conflicting result at (%d,%d) dir=%s success=%d
failure=%d\n",
                     x,
                     y,
                     direction_name(dir),
                     success_votes,
                     failure_votes);
        return false;
    }
private:
    DriverSession &driver_;
    int width_;
    int height_;
    int entry_x_;
    int entry_y_;
    int exit_x_;
    int exit_y_;
    uint32_t max_steps_;
    std::vector<std::vector<Cell>> cells_;
};
std::string PathToKeys(const std::string& path) {
    std::string keys;
    for (auto& ch: path) {


switch (ch)
        {
        case 'L':   keys.push_back('A');    break;
        case 'R':   keys.push_back('D');    break;
        case 'U':   keys.push_back('W');    break;
        case 'D':   keys.push_back('S');    break;
        default:    break;
        }
    }
    return keys;
}
static bool build_and_commit(DriverSession &driver, const MazeExplorer &explorer)
{
    std::vector<Direction> path;
    std::string flag;
    std::string moves;
    if (!explorer.shortest_path_to_exit(path)) {
        std::fprintf(stderr, "[ERROR] exit is unreachable in recovered map.\n");
        return false;
    }
    for (Direction dir : path) {
        moves.push_back(direction_char(dir));
    }
    std::printf("[PATH] shortest_length=%zu expected=%u\n", path.size(),
driver.maze_info().max_steps);
    std::printf("[PATH] shortest_path=%s\n", moves.c_str());
    std::printf("[PATH] shortest_path_keys=%s\n", PathToKeys(moves).c_str());
    std::printf("[PATH] path_hash=0x%08X\n", compute_path_hash(path));
    if (!driver.reset()) {
        return false;
    }
    for (Direction dir : path) {
        Observation obs;
        if (!driver.move(dir, obs)) {
            return false;
        }
        if (obs.result == MoveResult::Failure) {
            std::fprintf(stderr,
                         "[ERROR] shortest-path replay failed dir=%s source=%s result=%s\n",
                         direction_name(dir),
                         source_name(obs.source),
                         result_name(obs.result));
            return false;
        }
    }
    if (!driver.commit(path, flag)) {


std::fprintf(stderr, "[WARN] commit failed.\n");
        return false;
    }
    std::printf("[FLAG] %s\n", flag.c_str());
    return true;
}
int main()
{
    DriverSession driver;
    MazeExplorer *explorer = nullptr;
    bool ok = false;
    std::puts("KernelMaze Explorer");
    std::puts("-------------------");
    if (!driver.initialize()) {
        return 1;
    }
    std::printf("[*] maze: %ux%u start=(%u,%u) exit=(%u,%u) max_steps=%u\n",
                driver.maze_info().width,
                driver.maze_info().height,
                driver.maze_info().entry_x,
                driver.maze_info().entry_y,
                driver.maze_info().exit_x,
                driver.maze_info().exit_y,
                driver.maze_info().max_steps);
    explorer = new MazeExplorer(driver);
    ok = explorer->run();
    if (!ok) {
        delete explorer;
        return 1;
    }
    explorer->print_map();
    explorer->save_map("KernelMazeMap.txt");
    build_and_commit(driver, *explorer);
    delete explorer;
    return 0;
}
```

## 方法总结

- 核心技巧：逆向驱动入口、运行时字符串解密和 IOCTL 分发逻辑，把隐藏反馈通道恢复成可脚本化的迷宫探索接口。
- 识别信号：驱动题附带用户态客户端但反馈被刻意隐藏时，应重点检查驱动创建的设备名、符号链接、事件对象、注册表键和 IOCTL 状态返回。
- 复用要点：先恢复外层通信骨架，再抽象状态观测与移动接口；迷宫类题不要手工猜路径，应构建 BFS/DFS 探索器并用驱动原生提交逻辑验证。
