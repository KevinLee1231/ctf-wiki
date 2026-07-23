# YARA Trainer Gym

## 题目简述

网页允许上传任意文件，服务端用八条 YARA 规则扫描。每命中一条规则就击败一名道馆馆主，八条全部命中时返回 flag。官方仓库没有单独题解，但保留了完整规则、Flask 判定逻辑和一个用于构造样本的 ELF 源码，因此可以从实现还原目标条件。

八条规则要求上传文件同时满足：

1. 文件头是 ELF 魔数 `7f 45 4c 46`；
2. 同时包含 `jessie`、`james`、`meowth`；
3. ELF 节区数量等于 40；
4. 包含 `somethingsomethingmalware` 与 `TEAMROCKET`，或另一组两个字符串；
5. 整个文件的 Shannon 熵不低于 6；
6. 包含规则字符串 `aqvkpjmdofazwf{lqjm1310<` 的任意单字节 XOR 变体；
7. 存在名为 `poophaha` 的 ELF 节区；
8. 文件大小严格位于 1 MiB 与 2 MiB 之间。

## 解题过程

### 编译基础 ELF

仓库中的 `main.c` 已经放入大部分所需字符串，并用 GCC 属性创建自定义节区：

```c
int poop_var __attribute__((section("poophaha"))) = 42;

printf("jessie");
printf("james");
printf("meowth");
printf("TEAMROCKET");
printf("somethingsomethingmalware");
printf("bruhsinglebytexorin2023?");
```

最后一个字符串是规则 6 基准串逐字节异或 `0x03` 的结果。YARA 的 `xor` 修饰符会枚举单字节 XOR 密钥，所以它能命中。

编译时要防止优化器删除这些只为特征而存在的字符串：

```bash
gcc -O0 -o fighter main.c
```

### 调整节区数

用 `readelf -h fighter` 查看 `Number of section headers`。仓库中现成的 `main` 有 34 个节区，而规则要求恰好 40，因此再加入六个节区；若本机编译结果不同，就加入
$40-N$ 个，而不是照抄固定次数：

```bash
printf X > section-byte.bin

objcopy --add-section extra1=section-byte.bin fighter
objcopy --add-section extra2=section-byte.bin fighter
objcopy --add-section extra3=section-byte.bin fighter
objcopy --add-section extra4=section-byte.bin fighter
objcopy --add-section extra5=section-byte.bin fighter
objcopy --add-section extra6=section-byte.bin fighter

readelf -h fighter | grep "Number of section headers"
```

`poophaha` 节必须保留，不能用会重建或剥离全部自定义节区的后处理方式。

### 同时满足大小和熵

基础 ELF 只有约 16 KiB，熵也不足。最后追加约 1.2 MB 随机字节：

```bash
head -c 1200000 /dev/urandom >> fighter
```

ELF 加载器和 YARA 的 ELF 模块仍按头部节表解析文件，尾随字节不会改变 40 这个节区计数；但 YARA 的 `filesize` 和
`math.entropy(0, filesize)` 会覆盖这些随机数据，因此规则 5 与规则 8 同时满足。

提交前应在本地用仓库规则验证：

```bash
yara yara_rules.yar fighter
```

输出应包含 `rule1` 至 `rule8`。服务端以 `len(matches) == 8` 判定成功，返回：

```text
UMDCTF{Y0ur3_4_r34l_y4r4_m4573r!}
```

## 方法总结

- 核心技巧：把八条 YARA 条件拆成文件魔数、字符串、ELF 结构、熵和大小五类，再构造一个同时满足全部约束的合法 ELF。
- 识别信号：上传结果逐项显示规则命中状态时，可以把它当作约束 oracle，逐次只改变一个文件属性并观察反馈。
- 复用要点：追加随机数据能提高大小和全文件熵，却不能改变 ELF 节区数；结构条件必须用 `objcopy` 和 `readelf` 单独控制，最后再做 YARA 全量验证。
