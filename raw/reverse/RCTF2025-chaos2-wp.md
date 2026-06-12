# chaos2

## 题目简述

程序包含大量 `jnz` 到 `mov 22222222` 的花指令，并在多个小函数中做反调试和动态修改密钥字符。清理花指令后定位构造函数，恢复最终 RC4 密钥并解密。

## 解题过程

### 关键观察

程序包含大量 `jnz` 到 `mov 22222222` 的花指令，并在多个小函数中做反调试和动态修改密钥字符。

### 求解步骤

jnz到mov 22222222的都是花指令，全部nop
构造函数中检索了密钥的位置：
散装出现在各个小函数中的是反调试和动态修改：
int SetIndex_0x00_u()
{
  HMODULE ModuleHandleA; // [esp+4h] [ebp-8h]
  unsigned int n0x10000; // [esp+8h] [ebp-4h]

  ModuleHandleA = GetModuleHandleA(0);
  for ( n0x10000 = 0; n0x10000 < 0x10000; ++n0x10000 )
  {
    if ( *(_DWORD *)((char *)ModuleHandleA + n0x10000) == 0x12345678 && *
((_BYTE *)ModuleHandleA + n0x10000 + 4) != 'u' )
    {
      dword_D1440C = (int)ModuleHandleA + n0x10000 + 4;// "flag:
{Th1sflaglsG00ds}"
      // "flag:{Th1sflaglsG00ds}"
      return dword_D1440C;
    }
  }
  // "flag:{Th1sflaglsG00ds}"
  return dword_D1440C;
}
int SetIndex_0x08_i()
{
  int v1; // [esp+Ch] [ebp-Ch]
  uint8_t BeingDebugged; // [esp+13h] [ebp-5h]

  BeingDebugged = NtCurrentPeb()->BeingDebugged;
  SetIndex_0x00_u();
  index = 8;
  if ( !BeingDebugged )
    *(_BYTE *)(index + off_D1440C) = 'i';       // "flag:{Th1sflaglsG00ds}"
  return v1;
}

int __cdecl SetIndex_0x0E_I(int (__stdcall *a1)(HANDLE, int, int *, int,
_DWORD))
{
密钥最终变为flag:{ThisflagIsGoods} ，填充0x00到128字节，RC4解密即可
RCTF{AntiDbg_Reversing_2025_v2.0_Ch4llenge}
  int result; // eax
  HANDLE CurrentProcess; // [esp+Ch] [ebp-10h]
  int v3; // [esp+14h] [ebp-8h] BYREF

  CurrentProcess = GetCurrentProcess();
  result = a1(CurrentProcess, 7, &v3, 4, 0);
  index = 14;
  if ( v3 )
    off_D1440C[index] = 'I';                    // "flag:{Th1sflaglsG00ds}"
  return result;
}

HMODULE SetIndex_0x11_o()
{
  HMODULE hModule; // eax

  hModule = LoadLibraryW(L"Ntdll.dll");
  if ( hModule )
  {
    index = 0x11;
    hModule = (HMODULE)GetProcAddress(hModule, "NtClose");
    if ( hModule )
      return (HMODULE)((int (__stdcall *)(unsigned int))hModule)(0x99999999);
//异常处理，修改为o
  }
  return hModule;
}

int __cdecl SetIndex_0x12_o(int (__stdcall *a1)(HANDLE, int, int *, int,
_DWORD))
{
  int result; // eax
  HANDLE CurrentProcess; // [esp+Ch] [ebp-10h]
  int v3; // [esp+14h] [ebp-8h] BYREF

  CurrentProcess = GetCurrentProcess();
  result = a1(CurrentProcess, 31, &v3, 4, 0);
  index = 0x12;
  if ( v3 == 1 )
    off_D1440C[index] = 'o';                    // "flag:{Th1sflagIsGo0ds}"
  return result;
}

## 方法总结

- 核心技巧：去花指令、识别反调试动态改 key、RC4 解密。
- 识别信号：大量固定模式跳转、PEB/NtQueryInformationProcess/NtClose 异常检测。
- 复用要点：动态修改字符串时要按实际反调试分支恢复最终密钥。
