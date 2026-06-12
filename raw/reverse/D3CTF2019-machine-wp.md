# machine

## 题目简述

题目是 Android APK native 层逆向。`idleonce` 中通过 Ackermann 函数生成数组初值，数组后一个元素满足“前一个元素两倍加三”。`verify` 对应的 native 函数对 flag 前后两半分别做 XTEA 加密：前半 32 轮，密钥来自数组第 8 到 11 项；后半 64 轮，密钥包含多个反调试函数的返回值，其中无调试器时关键返回值为 1107。静态拿到两组密文和密钥后，按 XTEA 逆运算解密即可。

题目资料地址：https://github.com/pcy190/D3CTF-2019-Machine

题面描述强调这是一台“很慢的数学计算机器”，源码中可见 Android/Kotlin 主逻辑和 native 层重打包流程。native 函数通过 JNI 动态注册，`onload` 中包含 `ptrace`、进程搜索、Frida 检测、inotify 监控 maps 等反调试逻辑，多个返回值会参与密钥；无调试器状态下关键异或结果为 `1107`。Ackermann 部分实际使用 `ackman(3,16)`，当 `m=3,n>=5` 时可化为 `2^(n+3)-3`。

## 解题过程

分析apk的so文件。首先idleonce中通过ackerman函数生成array初始值。(可参见 [Ackermann function](https://en.wikipedia.org/wiki/Ackermann_function))。其生成的array，后一个元素是前一个元素的两倍+3.

跟入verify对应的native函数地址，这里其实是两次XTEA加密，分别对前半flag和后半flag进行加密。第一次是32轮，密钥由之前生成的array的第8-11个数组成，分别为2045,4093,8189,16381。第二次为64轮，密钥为1107，4096，524285，262141。其中，1107是多个反调试函数的返回值，直接静态分析不难发现如果没有检测到调试器，其返回值应该是1107。

静态分析可得，密文分别为

```
246553640 2138606322
-1227780298 -783437692
```

解密脚本

```cpp
void encrypt ( unsigned long* v, unsigned long* key ) {
    unsigned long l = v[0], r = v[1], sum = 0, delta = 0x9e3779b9;

    for ( size_t i = 0; i < 32; i++ ) {
        l += ( ( ( r << 4 ) ^ ( r >> 5 ) ) + r ) ^ ( sum + key[sum & 3] );
        sum += delta;
        r += ( ( ( l << 4 ) ^ ( l >> 5 ) ) + l ) ^ ( sum + key[ ( sum >> 11 ) & 3] );
    }

    v[0] = l;
    v[1] = r;
}

void decrypt ( unsigned long* v, unsigned long* key, unsigned int count ) {
    unsigned long l = v[0], r = v[1], sum = 0, delta = 0x9e3779b9;
    sum = delta * count;

    for ( size_t i = 0; i < count; i++ ) {
        r -= ( ( ( l << 4 ) ^ ( l >> 5 ) ) + l ) ^ ( sum + key[ ( sum >> 11 ) & 3] );
        sum -= delta;
        l -= ( ( ( r << 4 ) ^ ( r >> 5 ) ) + r ) ^ ( sum + key[sum & 3] );
    }

    v[0] = l;
    v[1] = r;
}

int main ( int argc, char const* argv[] ) {
    unsigned long v[2] = {0, 0};    
    unsigned long key[] = {2045, 4093, 8189, 16381};
    unsigned long key2[] = { 1107 ,4093, 524285 ,262141 };
    v[0] = 246553640 ;
    v[1] = 2138606322;
    decrypt ( v, key, 32 );
    for ( int i = 0; i < 8; i++ )
        printf ( "%c" , (( char* ) v)[i] );
    v[0] = -1227780298;
    v[1] = -783437692;
    decrypt ( v, key2, 64 );
    for (int i = 0; i < 8; i++)
        printf("%c", ((char*)v)[i]);
    return 0;
}
```

## 方法总结

- 核心技巧：从 native 校验函数中识别 XTEA 轮函数和 delta 常量，结合 Ackermann 生成数组恢复密钥，再分别按 32/64 轮解密两半 flag。
- 识别信号：APK so 中出现 `0x9e3779b9`、左右半轮移位异或加法、固定两字密文时，应优先判断 XTEA/TEA 家族。
- 复用要点：反调试函数返回值也可能参与密钥，静态分析时要分清“检测到调试器”和“正常运行”两条路径；有符号密文常量写进 C/C++ 解密脚本时要注意 32 位无符号转换。

