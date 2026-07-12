# Welcome to CTF

## 题目简述

题目是 Windows 逆向，难点集中在反调试、字符串/API 混淆、硬件断点改控制流和 MIRACL 大数库逻辑恢复。程序通过 VEH/UEF、硬件断点和多次 `NtSetInformationThread(0x11)` 影响异常传递与全局反调试结果；真实验证链为自定义码表 Base64 解码、RSA/MIRACL 大数运算、再拆成两个大数进入丢番图方程校验。最终需要构造满足方程的 x/z，RSA 加密后按魔改 Base64 表编码成 flag。

## 解题过程

### 主要流程和难点：

反调试，字符串混淆，部分函数指针混淆，api调用基本改成GetProcAddress(GetModuleHandleW(modulename),apiname)的形式，miracl大数库的逆向。

首先有四个反调试。

1.执行main函数之前，程序先注册VEH和UEF，并设置4个硬件断点，用来改变执行流程或者部分参数。也就是硬断占坑。

2.执行到硬断处后，VEH会设置T 标志位，产生单步异常后转发给UEF，在有调试器的状况下，异常不会转发到UEF（有SharpOD插件的OD可以过）。

3.设置完第三个硬断后，调用NtSetInformationThread 0x11。设置完所有硬断后，调用NtSetInformationThread 0x11，但是在该次调用，硬断会把第四个参数被设置为sizeof(PVOID)，正常情况该调用会失败，用带有StrongOD驱动或SharpOD插件的OD调试则会返回成功。

4.main函数会再调用一次NtSetInformationThread 0x11，是正常调用，让调试器接收不到异常。

第二次和第三次NtSetInformationThread的结果会存放到全局变量里，用于最后的校验。也就是说一开始被检测到调试器的话，后面就算输入正确的flag也是提示错误。

main函数没什么要点，判断输入是否为WMCTF{xxx}的形式，是则进入假的验证流程。一开始程序就在函数头设置了硬断，UEF会设置EIP到正确流程。

真实验证流程为：自定义码表base64解码->判断解码结果长度是否为96位->RSA解密->判断hex大数长度是否为29位->方程校验大数。

### 分析：

根据上面的主要流程，可以在SetUnhandledExceptionFilter函数下断，获取UEF的地址，或者直接在IDA里面找到调用SetUnhandledExceptionFilter的地方，推算出函数地址

```plain
  v9 = GetModuleHandleW(&v22);
  v10 = GetProcAddress(v9, &v16[1]);
  *v12 = v13(1, (char *)off_432F80 + 162259905);
  ((void (__stdcall *)(char *))v10)((char *)off_432F84 + 1987071348);
```
v10为SetUnhandledExceptionFilter函数地址， off_432F84处的值为89CFC4A8，加上key就是0x40121C。
UEF只调用了sub_402980，之后就返回了，sub_402980是根据异常触发的地址分别对异常进行处理。

```plain
int __thiscall sub_402980(_EXCEPTION_POINTERS *this)
{
  PCONTEXT v1; // edx
  char *v2; // esi
  _BYTE *v3; // edx
  signed int v4; // esi
  char v5; // cl
  DWORD v6; // eax
  v1 = this->ContextRecord;
  v2 = (char *)v1->Eip;
  if ( v2 == (char *)off_432F90 + 1505898260 )
  {
    v1->Eip = (DWORD)off_432F88 + 1505898259;
    this->ContextRecord->Esp += 4;
  }
  else if ( v2 == (char *)off_432F98 + 1923438580 )
  {
    sub_403F80(*(_DWORD *)((char *)off_432F8C + 1923438579), v1->Eax);
  }
  else if ( v2 == (char *)off_432F9C + 1654112553 )
  {
    v3 = (_BYTE *)(v1->Eax + 48);
    v4 = 5;
    do
    {
      v5 = *(v3 - 1);
      *(v3 - 1) = *v3;
      *v3 = v5;
      v3 += 2;
      --v4;
    }
    while ( v4 );
  }
  else if ( v2 == (char *)off_432F94 + 1918029978 && !byte_438C91 )
  {
    v6 = v1->Esp;
    byte_438C91 = 1;
    *(_DWORD *)(v6 + 20) = 4;
  }
  return 0;
}
```
第一个硬件断点下在了sub_4027C7，也就是假验证流程。对应的处理是修改eip为0x402885，伪代码如下。
```plain
char __cdecl sub_402885(int *a1)
{
  char v1; // bl
  _BYTE *v2; // eax
  _BYTE *v3; // esi
  int v4; // edx
  int v5; // ecx
  unsigned __int16 *v6; // edi
  _BYTE *v7; // edi
  unsigned __int16 *v9; // [esp+14h] [ebp-14h]
  unsigned __int16 *v10; // [esp+18h] [ebp-10h]
  int v11; // [esp+30h] [ebp+8h]
  v1 = 0;
  v2 = operator new[](0x100u);
  v3 = v2;
  v4 = a1[4];
  if ( (unsigned int)a1[5] >= 0x10 )
    v5 = *a1;
  if ( b64_decode(v2) == 96 )
  {
    v6 = (unsigned __int16 *)mirvar(0);
    v11 = (int)v6;
    v9 = (unsigned __int16 *)mirvar(0);
    v10 = (unsigned __int16 *)mirvar(0);
    bytestobig(v3, 96, v6);
    RSA(v6);
    if ( numdig(v6) == 29 )
    {
      v7 = operator new[](0x100u);
      bigtobytes(v11, v7);
      bytestobig(v7, 8, v9);
      bytestobig(v7 + 8, 7, v10);
      v1 = check(v9, v10);
      if ( v7 )
        j_j__free(v7);
      v6 = (unsigned __int16 *)v11;
    }
    mirkill(v6);
    mirkill(v9);
    mirkill(v10);
  }
  if ( v3 )
    j_j__free(v3);
  return v1;
}
```
base64部分魔改了码表和码表的形式，UEF还会调换部分码表的顺序，真实码表为
 `{255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 9, 255,             255, 255, 54, 63, 8, 46, 12, 4, 27, 10, 0, 39, 47, 255, 255,             255, 255, 255, 255, 255, 29, 53, 41, 11, 17, 59, 5, 49, 21, 7,             16, 35, 40, 2, 38, 24, 55, 30, 60, 26, 50, 34, 6, 31, 45,             52, 255, 255, 255, 255, 255, 255, 25, 23, 48, 58, 19, 44, 15, 62,             51, 56, 13, 28, 1, 18, 20, 22, 61, 14, 32, 43, 57, 37, 3,             42, 33, 36, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,             255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255}`

可以写代码还原成可见字符串。码表为 7mNw4GWJ1+6D3krgKEneoIpbPaT5lARXsyVLzvO8MCxtfY29cHUiZB/QjudFSqh0 ，直接用上面那个表也没问题。base64解码结果转成大数，进入RSA部分。

RSA部分没有硬断，可以提取出d和n,n是已经被分解出来的RSA-768,直接在网上搜索得到p q

，进而求出e。解密出来的结果会转成bytes，前8个字节和后7个字节分别转成大数。进入校验部分。

```plain
char __thiscall RSA(void *this)
{
  void *v1; // ebx
  _BYTE *v2; // eax
  signed int v3; // edi
  _BYTE *v4; // esi
  unsigned __int16 *v6; // [esp+Ch] [ebp-74h]
  unsigned __int16 *v7; // [esp+10h] [ebp-70h]
  void *v8; // [esp+14h] [ebp-6Ch]
  char v9; // [esp+18h] [ebp-68h]
  v8 = this;
  v1 = (void *)mirvar(0x10000);
  v7 = (unsigned __int16 *)mirvar(0);
  v6 = (unsigned __int16 *)mirvar(0);
  v2 = (_BYTE *)getN((int)&v9);
  v3 = 96;
  v4 = v2 + 1;
  do
  {
    *v4++ ^= *v2;
    --v3;
  }
  while ( v3 );
  v2[97] = 0;
  bytestobig(v2 + 1, 96, v7);
  incr(1, (int)v1);
  powmod((int)v1, (int)v8, (int)v7, v6);
  copy(v6, v8);
  mirkill(v1);
  mirkill(v7);
  mirkill(v6);
  return 1;
}
```
大数校验部分，是一道丢番图方程，也是可以在网上找到答案，为了提示，方程已经内置了一个大数，比较结果时，会减去反调试结果(反调试结果记为anti)。大致为
(-x)^3+(80435758145817515)^3+ (z)^3 = 43-anti

正常运行时anti为1，方程右边为42，用 80435758145817515 42等字样搜索，就能得到答案，x=80538738812075974，z=12602123297335631。

按照上面的流程，将x和z转成16进制，拼接到一起后，RSA加密后base64，补上WMCTF{}就是flag了。

附上py脚本

```python
from string import maketrans
import gmpy2

def getb64strtable():
    b64table = [
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 9, 255,
        255, 255, 54, 63, 8, 46, 12, 4, 27, 10, 0, 39, 47, 255, 255,
        255, 255, 255, 255, 255, 29, 53, 41, 11, 17, 59, 5, 49, 21, 7,
        16, 35, 40, 2, 38, 24, 55, 30, 60, 26, 50, 34, 6, 31, 45,
        52, 255, 255, 255, 255, 255, 255, 25, 23, 48, 58, 19, 44, 15, 62,
        51, 56, 13, 28, 1, 18, 20, 22, 61, 14, 32, 43, 57, 37, 3,
        42, 33, 36, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255,
        255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255, 255
    ]
    strt = [''] * 64
    for i in range(len(b64table)):
        if b64table[i] != 255:
            strt[b64table[i]] = chr(i)
    return ''.join(strt)

if __name__ == '__main__':
    tmp_m = '011E218E658D3FC62CC5907A8DA94F'  # 11E218E658D3FC6+2CC5907A8DA94F
    # tmp_e = '740DE48760442835BAAD5E1990453A9D16DB7976D3F8BB98BF99C0C01CBE9B9C12B808C80683D1E346C16C79AC162874F28CA610C1B97E5E1FFAE95725CE0C6B031C3E188B17187A793B322CC4004C568E76C9B258542EA2A2D6ECD462FFF401'
    tmp_n = 'CAD984557C97E039431A226AD727F0C6D43EF3D418469F1B375049B229843EE9F83B1F97738AC274F5F61F401F21F1913E4B64BB31B55A38D398C0DFED00B1392F0889711C44B359E7976C617FCC734F06E3E95C26476091B52F462E79413DB5'
    b64_s = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
    b64_t = getb64strtable()
    # b64_t = '7mNw4GWJ1+6D3krgKEneoIpbPaT5lARXsyVLzvO8MCxtfY29cHUiZB/QjudFSqh0'
    tran = maketrans(b64_s, b64_t)
    p = 33478071698956898786044169848212690817704794983713768568912431388982883793878002287614711652531743087737814467999489
    q = 36746043666799590428244633799627952632279158164343087642676032283815739666511279233373417143396810270092798736308917
    d = 0x10001
    m = int(tmp_m, 16)
    # e = int(tmp_e, 16)
    phiN = (p - 1) * (q - 1)
    e = gmpy2.invert(d, phiN)
    print 'e =', hex(e)
    n = int(tmp_n, 16)
    enc = pow(m, e, n)
    dec = pow(enc, d, n)
    print 'test:', hex(dec)  # test RSA
    b64 = hex(enc)[2:]
    b64 = b64.decode('hex')
    flag = 'WMCTF{' + b64.encode('base64').translate(tran).replace('\n', '') + '}'
    print len(b64), flag
```

## 方法总结

- 核心技巧：通过 UEF/VEH 恢复被硬件断点改写的真实控制流，提取魔改 Base64 表、MIRACL RSA 参数和方程校验参数，反向构造合法输入。
- 识别信号：程序在 main 前注册异常处理、设置硬件断点、API 通过 `GetProcAddress(GetModuleHandleW(...), ...)` 动态解析，且调试器会影响最终校验变量时，应先还原异常分发逻辑再看伪代码。
- 复用要点：反调试结果参与最终校验，不能只 patch 掉前置检测；本题正常运行时 `anti=1`，方程右侧为 `42`，搜索内置大数可定位已知 Ramanujan/Nagell 类方程解。
