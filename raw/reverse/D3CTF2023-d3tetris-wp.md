# d3Tetris

## 题目简述

题目是 Android APK + native 库逆向。抓包可见 `application/x-protobuf` 请求，需要先恢复/解析 protobuf 字段，再转入 JNI 生成字段的 `libnative.so`。native 层包含 anti-Frida、自解密字符串、改造 AES 和 RC4；绕过检测并拿到 key 后，按修改后的 AES 轮函数顺序解密恢复 flag。

## 解题过程

题目给了一个数据包，用 Wireshark 打开后可以看到一个 POST 请求，其 `content-type` 为 `application/x-protobuf`。

使用 jadx 反编译 APK，然后在 jadx 中搜索 `application/x-protobuf`，即可找到 HTTP 请求相关逻辑。

关键源码片段显示题目会把请求参数写入配置或脚本上下文，后续分析集中在这些可控字段如何进入校验逻辑。

要反序列化请求数据，可以先逆向并恢复 `.proto` 文件，再用 protoc 反序列化；也可以直接使用 `blackboxprotobuf` 解析数据。

反序列化请求数据后，可以发现两个可疑字段，它们的内容由 JNI native 函数生成，因此转向分析 `libnative.so`。

`libnative.so` 中在 `.init_array` 加入了 anti-Frida 逻辑，例如检测 Frida 线程、pipename 和内存扫描。参考的 DetectFrida 实现有用之处在于展示了常见 native anti-Frida 检查：扫描线程名、检查命名管道/maps、在进程内存中搜索 Frida 指纹。本题中这些检查会在正常 JNI 逻辑前运行，所以 hook 必须在 `.init_array` constructor 执行前安装。

为了对抗 anti-Frida，可以在 `.init_array` 中函数执行前先 hook，脚本如下：

```
function hook_init_array(module_name) {
    if (Process.pointerSize == 4)
        var linkername = "linker";
    else if (Process.pointerSize == 8)
        var linkername = "linker64";
    var symbols = Module.enumerateSymbolsSync(linkername);
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        if (symbol.name.indexOf("call_constructor") !== -1) {
            var call_constructor_addr = symbol.address;
        }
    }
    if(call_constructor_addr.compare(NULL) > 0) {
        console.log("get construct address");
        Interceptor.attach(call_constructor_addr, {
            onEnter: function(args) {
                if(module_name){
                    const tagetModule = Process.findModuleByName(module_name);
                    if(tagetModule){
                        console.log("hook: "+module_name);
                        module_name = null;
                        hook_anti_frida();
                    }
                }
            },
            onLeave: function(retval) {
            }
        });
    }
}
function hook_anti_frida(){
    var base = Module.findBaseAddress("libnative.so");
    // 64 bit version as example
    var antiFridaFunc = base.add(0x13900);
    Interceptor.replace(antiFridaFunc, new NativeCallback(function (arg0) {
        return;
    }, "void", ["void"]));
}
hook_init_array("libnative.so");
```

第一个 JNI 函数会获取设备的 bootid（每次设备重启都会更新），然后使用修改版 AES 和未修改版 RC4 对其加密。

程序会读取 `/proc/sys/kernel/random/boot_id`，把 boot id 作为派生随机数或密钥材料的一部分。

第二个 JNI 函数会获取设备序列号，并把它作为 AES 加密的 IV 向量。

`libnative` 中的字符串都经过加密，使用前会解密，随后再重新加密，以防止直接通过内存 dump 还原。但 Frida 检测已经被

绕过，因此可以通过 Frida 获取明文字符串，并拿到 AES 与 RC4 加密密钥。

AES 加密被修改过，`mixcolumn` 和 `shiftrows` 的顺序被交换，因此解密时也需要调整顺序。解密部分的修改如下：

```
// ...
// sbox 已修改
const unsigned char inv_sbox[16][16] = {
    {0x4b, 0x26, 0x7c, 0xf2, 0x48, 0xfd, 0x61, 0xd1, 0x16, 0x91, 0xe2, 0x89,
0x1f, 0xa7, 0x8c, 0xed },
{0x8f, 0xd2, 0x9b, 0x28, 0x81, 0xc9, 0x19, 0xde, 0x4a, 0x2f, 0xce, 0xf4, 0x86,
0xa3, 0x47, 0xdd },
{0x9a, 0x0d, 0xb5, 0x0a, 0xf9, 0xe0, 0x57, 0x2e, 0x27, 0xac, 0xba, 0x88, 0xdc,
0x38, 0x75, 0x06 },
{0xab, 0x64, 0x73, 0xd3, 0x09, 0xa2, 0xc3, 0x78, 0x84, 0xda, 0xd8, 0x7e, 0xd0,
0x10, 0xe3, 0x62 },
{0xc8, 0x76, 0xbd, 0x4f, 0xb3, 0xfc, 0x87, 0xc4, 0x1a, 0x34, 0x39, 0xff, 0x83,
0xe9, 0xf7, 0x0c },
{0x02, 0x50, 0xa5, 0x45, 0x5e, 0xb9, 0xbf, 0x35, 0x3b, 0x1d, 0x53, 0x5b, 0xdb,
0xca, 0xb7, 0xc1 },
{0x65, 0xb2, 0x8e, 0xf6, 0x9c, 0xe7, 0x0e, 0xfb, 0x3d, 0x15, 0xe1, 0x0f, 0x2a,
0x96, 0xd5, 0x90 },
{0xa0, 0xf1, 0x8b, 0x59, 0xe6, 0xb6, 0x7f, 0x7b, 0xec, 0x4e, 0x01, 0x41, 0x93,
0x07, 0xae, 0x18 },
{0x7d, 0x25, 0x6e, 0xb0, 0x52, 0x67, 0xc6, 0x36, 0x56, 0x85, 0x2d, 0xad, 0x44,
0x74, 0xcb, 0x92 },
{0x00, 0xbb, 0x80, 0xe4, 0x40, 0xb1, 0xd7, 0x55, 0xfa, 0x33, 0xa4, 0x98, 0x05,
0x5a, 0xd6, 0x6f },
{0x08, 0x66, 0xbe, 0xb4, 0x31, 0xc7, 0xaa, 0xb8, 0x22, 0xf3, 0x42, 0xe8, 0x99,
0x46, 0x6c, 0xa8 },
{0x51, 0x60, 0xee, 0x13, 0x3c, 0xc0, 0x58, 0x79, 0x29, 0x24, 0xa9, 0xd4, 0x63,
0xcc, 0xc5, 0x77 },
{0xaf, 0x3a, 0x12, 0x6d, 0x8a, 0x49, 0x6a, 0x3f, 0xcd, 0x68, 0x0b, 0x2b, 0x9e,
0x8d, 0x71, 0xea },
{0x82, 0xcf, 0x30, 0x32, 0x94, 0x1b, 0xf5, 0x95, 0x1e, 0xf8, 0xd9, 0xc2, 0x2c,
0x9d, 0x3e, 0x37 },
{0x11, 0xef, 0x43, 0xfe, 0x21, 0x5d, 0xdf, 0x6b, 0xeb, 0xe5, 0x23, 0x1c, 0x7a,
0x4c, 0x54, 0x03 },
{0x04, 0x4d, 0xbc, 0x20, 0x5f, 0xa6, 0xf0, 0x97, 0x69, 0xa1, 0x9f, 0x70, 0x14,
0x72, 0x5c, 0x17 }};
// ...
// AesUtils.cpp
void AES::EncryptBlock(const unsigned char in[], unsigned char out[],
                       unsigned char *roundKeys) {
  unsigned char state[4][Nb];
  unsigned int i, j, round;
  for (i = 0; i < 4; i++) {
    for (j = 0; j < Nb; j++) {
      state[i][j] = in[i + 4 * j];
    }
  }
  AddRoundKey(state, roundKeys);
  for (round = 1; round <= Nr - 1; round++) {
    SubBytes(state);
    MixColumns(state);    // 顺序已交换
    ShiftRows(state);
    AddRoundKey(state, roundKeys + round * 4 * Nb);
  }
  SubBytes(state);
  ShiftRows(state);
  AddRoundKey(state, roundKeys + Nr * 4 * Nb);
  for (i = 0; i < 4; i++) {
    for (j = 0; j < Nb; j++) {
      out[i + 4 * j] = state[i][j];
    }
  }
}
void AES::DecryptBlock(const unsigned char in[], unsigned char out[],
                       unsigned char *roundKeys) {
  unsigned char state[4][Nb];
  unsigned int i, j, round;
  for (i = 0; i < 4; i++) {
    for (j = 0; j < Nb; j++) {
      state[i][j] = in[i + 4 * j];
    }
  }
  AddRoundKey(state, roundKeys + Nr * 4 * Nb);
  InvShiftRows(state);
  InvSubBytes(state);
  AddRoundKey(state, roundKeys + (Nr - 1) * 4 * Nb);
  InvShiftRows(state);
  for (round = Nr - 2; round >= 1; round--) {
    InvMixColumns(state);
    InvSubBytes(state);
    AddRoundKey(state, roundKeys + round * 4 * Nb);
    InvShiftRows(state);
  }
  InvMixColumns(state);
  InvSubBytes(state);
  AddRoundKey(state, roundKeys);
  for (i = 0; i < 4; i++) {
    for (j = 0; j < Nb; j++) {
      out[i + 4 * j] = state[i][j];
    }
  }
}
```

最终验证脚本中使用固定 `real_key` 和 `cipher`，通过 AES 解密得到结果字符串。

由于更新前的附件只加密了前 32 字节，导致解出的 flag 缺少 4 个字符，给所有参赛者带来了不便，这里表示歉意。

## 方法总结

- 核心技巧：protobuf 流量还原、JADX 定位 JNI 调用、hook linker constructor 绕过 `.init_array` anti-Frida、运行时抓明文字符串和 key、修改版 AES/RC4 解密。
- 识别信号：APK 请求体是 protobuf 且关键字段来自 native 函数时，静态解析 Java 只能定位入口，真正算法通常在 `.so`；若字符串会解密后再加密回去，应动态 hook 取瞬时明文。
- 复用要点：外链 anti-Frida 代码的必要信息是检测类别和执行时机；本题绕过点不是改每个检测，而是在 linker 调 constructor 前插入 hook。
