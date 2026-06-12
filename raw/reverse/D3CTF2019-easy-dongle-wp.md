# easy_dongle

## 题目简述

题目模拟“加密狗”保护：上位机是 32 位 ELF，内含被 DES-CBC 加密的目标 ELF；下位机是 STM32F103C8T6 固件，通过 UART 协议与上位机通信。上位机会校验单片机 UID，再把加密数据按 1025 字节封包发送给固件解密，最后用 ELF loader 加载解密结果执行。解题需要同时逆 ELF、STM32 固件加载方式、串口协议和 DES 参数；由于被加密程序本身直接打印 flag，解出第二块明文即可看到结果。

## 解题过程

主要思路是实现了个简单（且不完善）的加密狗壳，主要的点还是firmware和elf文件结合似乎之前比赛中没怎么看到过

上位机可执行文件：target （ELF-32文件）

下位机可执行文件：dongle.bin （bin文件）

下面按题目的运行流程说明固件加载、串口协议和解密脚本。

#### bin文件加载方式

下位机型号：STM32F103C8T6

处理器： ARM Little Endian

指令集： ARMv7-M（在IDA加载时注意指定）

内存映射：

- flash基址0x08000000，大小64K（64K是芯片flash大小，实际create segment时的size参数默认就行）
- RAM基址0x20000000，大小20K
- 加载地址0x08000000

IDA 加载 STM32 裸 `bin` 时，可以参考 STM32L0x 的 proc map [cfg 文件](https://github.com/joestanding/ida-stm32l-proc-map) 或 STM32F1 辅助 [脚本](https://github.com/screwer/IDA-scripts/blob/master/STM32/ida_stm32f1xx.py)。这些工具的作用不是直接解题，而是帮 IDA 建好 flash/RAM segment、外设地址映射和中断向量表入口；真正需要确认的是加载基址 `0x08000000`、RAM `0x20000000`、ARMv7-M 指令集，以及 reset handler 之后进入的主逻辑。题目提示“注意内存映射、指令集和中断向量表”，就是为了把分析起点收敛到这些参数。

#### 工作原理大致如下

- 加壳过程： target本身是个和下位机（STM32单片机）通讯的程序，要被加密的ELF文件被加壳器DES加密后放在target的dongle段
- target访问串口设备从而连接到下位机上，并判断下位机是否为唯一指定的设备（通过单片机的uid）。尔后经过一个简单的协议发送加密的数据到单片机上DES解密，并发回，target程序用一个类似elf loader的程序从发回的内存中加载执行被加密ELF（这里的被加密ELF是个简单的print flag的程序）

然后题目里被加密的程序是个简单的printf(flag)程序，所以flag在第二块1024字节解密时即可看到 = =，这也算是这题的漏洞吧，应该改成一个简单的xor flag程序的。不过其实这题在设计之初点也不在这，所以保留了elf loader程序的一些字符串。这段 loader 改自 [tel_ldr 的 elf-clean.c](https://github.com/shinh/tel_ldr/blob/e5bf6b8e060f7932c904ea38d0aa6d61f014b543/elf-clean.c)，核心逻辑是遍历 ELF program header，把 `PT_LOAD` 段按 `p_vaddr/p_offset/p_filesz/p_memsz` 映射到内存，处理完入口地址后跳转执行；本题只需要理解它会从单片机返回的明文中加载内嵌 ELF。

#### 协议及流程

本来想写个函数地址及其对应功能，不过感觉这样搞wp又臭又长  
大概说下流程：

##### UART通讯

首先ELF端数据接收主要还是用read/write系统调用，但单片机端实现了一个循环队列（源码在bsp/serial.c 不知道有没有人被这段坑到），主要结构体如下

```cpp
typedef struct
{
    USART_TypeDef* usart;
    uint32_t begin;
    uint32_t count;
    uint8_t *buf;
} usart_buf_ctr;
```

队列逻辑主要在串口中断处理程序里实现。

##### 上层协议封装

在串口通讯基础上实现了一个简单的协议，由协议头和内容组成，协议头结构体如下

```cpp
typedef struct
{
    conn_type type;
    uint16_t length;
    uint16_t crc16;
} conn_header;
```

发送和接收时都是先发送头，再根据length字段发送数据，最后校验crc16

##### 通信协议

```rust
master                     slave（初始等待接收）
握手包(conn_type=0xa5)->
                            <-确认包（conn_type=0x5a）
确认包(conn_type=0x5a)->                                   //类似tcp三次握手，这几个包的length属性都为0
                            <-字符串"easy dongle V0.1"
字符串"ok" ->
                            <-单片机8字节uid
异或后的8字节uid，作为密钥->
                            <-字符串"ok"
//=======接下来开始进行解密流程=========
以1025字节为一个封包传送加密数据
其中数据的第一字节为1时表示进行解密操作
为2时表示解密结束
                            <-每个1025字节封包解密后的数据
//==================================
发送第一个字节为2的包    ->
                            <-字符串"ok"，代表解密结束
```

##### 解密算法

简单的des-cbc，padding为PKCS5，IV为”D3CTF{0}”

```python
import os
import struct
import pyDes

def get_target_bin(off):
    target_bin = ""
    binlen = 0
    with open("easy_dongle.elf") as f:
        f.seek(off)
        target_bin = f.read()
        binlen = struct.unpack("<l", target_bin[:4])[0]
        print "binary file len: %d" %binlen
    return target_bin[4:4+binlen]

def gen_decryption(x):
    key = "\xce\x05\xc6\xa1\x1e\x0c\x2a\xee"
    iv = "D3CTF{0}"
    pack_size = 1025 - 1
    out = ""
    xlen = len(x)
    print "len x = %d" %xlen
    for i in xrange(0, xlen, pack_size):
        if xlen-i > pack_size:
            end = i + pack_size
        else:
            end = xlen

        k = pyDes.des(key, pyDes.CBC, iv, pad=None, padmode=pyDes.PAD_PKCS5)
        tmp = k.decrypt(x[i:end])
        out += tmp
        print "len(out) = %d" %len(out)
    return out

if __name__ == '__main__':
    target_bin = get_target_bin(0x3108)
    with open("elf_out", "wb") as f:
        tmp = gen_decryption(target_bin)
        f.write(tmp)
    os.system("chmod +x elf_out")
    os.system("./elf_out")
```

## 方法总结

- 核心技巧：把上位机 ELF 和下位机固件作为一个协议系统分析，先还原 UART 封包、UID 派生密钥和 DES-CBC 参数，再离线解密内嵌 ELF。
- 识别信号：题目同时给 ELF 和裸 `bin` 固件、出现串口读写和 MCU UID 校验时，应按“加密狗/授权设备”模型分析通信协议。
- 复用要点：STM32 固件导入 IDA 时要设置 ARMv7-M、flash/RAM 基址和向量表；协议层先头部再数据再 CRC 的结构有利于切出真实密文块。

